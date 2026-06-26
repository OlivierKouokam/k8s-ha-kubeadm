#!/usr/bin/env bash
# ============================================================
# common_script.sh — Cas 1 : HAProxy + Keepalived sur masters
#
# Usage (injecté par Vagrant) :
#   master : $1=master  $2=NODE_IP  $3=VIP  $4=IFACE  $5=VRRP_ID  $6=PRIORITY  $7=MASTER_IPS (csv)
#   worker : $1=worker  $2=NODE_IP  $3=VIP
# ============================================================

set -euo pipefail

ROLE="${1}"
NODE_IP="${2}"
VIP="${3}"

# ── Paramètres communs ──────────────────────────────────────
K8S_VERSION="1.30"
POD_CIDR="10.244.0.0/16"
HAPROXY_PORT="8443"

log() { echo "[$(date '+%H:%M:%S')] $*"; }

# ============================================================
# 1. Prérequis système (tous les nœuds)
# ============================================================
log "==> Prérequis système"

# Désactiver swap
swapoff -a
sed -i '/swap/d' /etc/fstab

# Modules noyau
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter

# Paramètres sysctl
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system -q

# ============================================================
# 2. Containerd
# ============================================================
log "==> Containerd"

apt-get update -qq
apt-get install -y -qq ca-certificates curl gnupg lsb-release

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  > /etc/apt/sources.list.d/docker.list

apt-get update -qq
apt-get install -y -qq containerd.io

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd
systemctl enable containerd

# ============================================================
# 3. kubeadm / kubelet / kubectl
# ============================================================
log "==> kubeadm + kubelet + kubectl"

apt-get install -y -qq apt-transport-https
curl -fsSL "https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/Release.key" \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v${K8S_VERSION}/deb/ /" \
  > /etc/apt/sources.list.d/kubernetes.list

apt-get update -qq
apt-get install -y -qq kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Configurer l'IP du nœud pour kubelet
echo "KUBELET_EXTRA_ARGS=--node-ip=${NODE_IP}" > /etc/default/kubelet
systemctl restart kubelet

# ============================================================
# 4. HAProxy + Keepalived (masters uniquement)
# ============================================================
if [[ "${ROLE}" == "master" ]]; then

  IFACE="${4}"
  VRRP_ID="${5}"
  PRIORITY="${6}"
  MASTER_IPS_CSV="${7}"

  IFS=',' read -ra MASTER_IPS <<< "${MASTER_IPS_CSV}"

  log "==> HAProxy + Keepalived (priority=${PRIORITY})"

  apt-get install -y -qq haproxy keepalived

  # ── HAProxy ─────────────────────────────────────────────
  cat > /etc/haproxy/haproxy.cfg <<EOF
global
    log /dev/log local0
    maxconn 4096

defaults
    mode tcp
    log global
    timeout connect 5s
    timeout client  30s
    timeout server  30s

frontend k8s-api
    bind *:${HAPROXY_PORT}
    default_backend k8s-masters

backend k8s-masters
    balance roundrobin
    option tcp-check
EOF

  for ip in "${MASTER_IPS[@]}"; do
    SNAME="master-$(echo "${ip}" | awk -F. '{print $NF}')"
    echo "    server ${SNAME} ${ip}:6443 check" >> /etc/haproxy/haproxy.cfg
  done

  systemctl enable haproxy
  systemctl restart haproxy

  # ── Keepalived ──────────────────────────────────────────
  # Détermination du state initial
  STATE="BACKUP"
  [[ "${PRIORITY}" == "110" ]] && STATE="MASTER"

  cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id k8s_ha
    enable_script_security
}

vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight   2
}

vrrp_instance VI_1 {
    state             ${STATE}
    interface         ${IFACE}
    virtual_router_id ${VRRP_ID}
    priority          ${PRIORITY}
    advert_int        1

    authentication {
        auth_type PASS
        auth_pass k8sha
    }

    virtual_ipaddress {
        ${VIP}
    }

    track_script {
        chk_haproxy
    }
}
EOF

  systemctl enable keepalived
  systemctl restart keepalived

fi

# ============================================================
# 5. Hosts resolution
# ============================================================
log "==> /etc/hosts"

# À adapter selon vos IPs si besoin
cat >> /etc/hosts <<EOF
192.168.80.10  k8s-vip
192.168.80.11  master1
192.168.80.12  master2
192.168.80.13  master3
192.168.80.21  worker1
192.168.80.22  worker2
EOF

log "==> Nœud ${ROLE} (${NODE_IP}) prêt"
