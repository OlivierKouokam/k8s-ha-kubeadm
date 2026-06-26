sudo kubeadm reset -f
sudo rm -rf /etc/kubernetes/ /var/lib/kubelet/ /var/lib/etcd/ /etc/cni/net.d

# ============================================================
# NOTES : étapes post-provisioning à exécuter manuellement
# ============================================================
# Sur master1 :
  sudo kubeadm init \
    --control-plane-endpoint "192.168.80.10:8443" \
    --upload-certs \
    --pod-network-cidr 10.244.0.0/16 \
    --apiserver-advertise-address 192.168.80.11
#
# Installer CNI (ex: Flannel) :
#   kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
#
# Rejoindre master2/master3 (commande issue de kubeadm init) :
  kubeadm join 192.168.80.10:8443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash> \
    --control-plane --certificate-key <key> \
    --apiserver-advertise-address <ip_du_master>
#
# Rejoindre les workers :
  kubeadm join 192.168.80.10:8443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>


# Opérateur Tigera
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/tigera-operator.yaml

# Télécharger la configuration custom pour vérifier le podCIDR
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.29.0/manifests/custom-resources.yaml

# S'assurer que le CIDR correspond à notre podSubnet 10.244.0.0/16
grep cidr custom-resources.yaml
Si différent de 10.244.0.0/16 :
sed -i 's|192.168.0.0/16|10.244.0.0/16|' custom-resources.yaml

kubectl create -f custom-resources.yaml

# Vérifier que les pods sont bien en Running sur tous les noeuds
kubectl get pods -n kube-system -o wide | grep -E "api|controller|scheduler|etcd"

# Vérifier l'état du cluster Etcd (sur master1) :
kubectl exec -it -n kube-system etcd-master1 -- /bin/sh -c "ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
endpoint health --cluster"

# Vérifier la liste des membres Etcd (sur master1) :
kubectl exec -it -n kube-system etcd-master1 -- /bin/sh -c \
"ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
member list -w table"


# Déployer une application de test (ex: webapp-color) pour vérifier la haute disponibilité
kubectl create deploy wbcolor --image kodekloud/webapp-color --replicas 3
kubectl expose deploy wbcolor --type NodePort --port 8080 --target-port 8080


# install baremetal loadbalancer : metallb and nginx ingress controller
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
kubectl get all -n metallb-system
# kubectl apply -f https://raw.githubusercontent.com/eazytraining/kubernetes-certified-administrator-training/v1.29/09_lab-09/ip_address_pool/ipaddresspool.yml
kubectl apply -f /home/vagrant/ipaddresspool.yml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml

kubectl get svc -n ingress-nginx -o wide
# get external IP of ingress-nginx-controller service loadbalancer 

kubectl patch svc wbcolor -p '{"spec": {"type": "LoadBalancer"}}'

git clone https://github.com/OlivierKouokam/example-voting-app.git
cd example-voting-app/k8s-specifications/
kubectl apply -f .

## CRASH / CHAOS TEST

# 1. Lancer une surveillance des nœuds en temps réel
watch kubectl get nodes

# 2. (Optionnel) Si vous avez déployé un pod de test, lancez une boucle de requêtes HTTP
# pour prouver qu'il n'y a aucune coupure de service
while true; do curl -s http://<IP_VIP_OU_APP> | grep -i "title"; sleep 0.5; done

# En tant que root, on force l'extinction immédiate (simulation de coupure de courant)
shutdown -h now

# 3. Observer le changement d'état. master01 passe "NotReady", mais les commandes répondent toujours !
kubectl get nodes

# 4. Vérifier quel Load Balancer (HAProxy/Keepalived) a récupéré la VIP (à taper sur master02 ou master03)
ip addr show

# Installation du binaire etcdctl pour effectuer un backup/restore du cluster Etcd
# 1. Définir la version souhaitée
ETCD_VER=v3.5.11

# 2. Télécharger l'archive officielle
curl -L https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd.tar.gz

# 3. Extraire uniquement le binaire etcdctl dans /usr/local/bin
sudo tar -xvf /tmp/etcd.tar.gz -C /usr/local/bin --strip-components=1 etcd-${ETCD_VER}-linux-amd64/etcdctl

# 4. Nettoyer l'archive
rm -f /tmp/etcd.tar.gz

# 1. Prendre le snapshot Etcd en passant les certificats de sécurité
sudo ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /tmp/etcd-backup-ha.db

# 2. Vérifier que le fichier de sauvegarde est valide
sudo ETCDCTL_API=3 etcdctl --write-out=table snapshot status /tmp/etcd-backup-ha.db

# 3. Supprimer volontairement des éléments importants ou tester une panne
kubectl delete deployment --all --all-namespaces
kubectl get deployments --all-namespaces # Constater le vide

# 4. Arrêter temporairement le manifeste de l'API Server et d'Etcd (en déplaçant les fichiers)
# >>>>>>>>>>>>>>>> (Faire cela sur TOUS les masters) <<<<<<<<<<<<<<<<<<
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 5. Restaurer le snapshot dans un nouveau répertoire de données (Sur master01)
sudo ETCDCTL_API=3 etcdctl \
  --data-dir=/var/lib/etcd-from-backup \
  snapshot restore /tmp/etcd-backup-ha.db

# 6. Supprimer l'ancien répertoire de données Etcd pour 
#      forcer la restauration sur >>>>>> master02 & master03 <<<<<<<
sudo rm -rf /var/lib/etcd/*

# 6. Modifier la configuration d'etcd.yaml pour pointer vers le nouveau dossier /var/lib/etcd-from-backup
# (Montrer à l'écran la modification du volume "etcd-data" dans /tmp/etcd.yaml)
sudo sed -i 's|/var/lib/etcd|/var/lib/etcd-from-backup|g' /tmp/etcd.yaml

# 7. Remettre les manifestes en place pour relancer les services
# >>>>>>>>>>>>> (Faire cela sur tous les masters une fois leurs dossiers etcd synchronisés ou configurés) <<<<<<<<<<<<<
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 8. Vérifier que les pods redémarrent et que les données supprimées sont revenues
kubectl get pods -n kube-system
kubectl get deployments --all-namespaces

