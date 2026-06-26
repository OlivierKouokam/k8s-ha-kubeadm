# -*- mode: ruby -*-
# vi: set ft=ruby :

# ============================================================
# Cluster Kubernetes HA — Cas 1 : HAProxy + Keepalived sur masters
# Topologie : 3 masters (stacked etcd + LB) + 2 workers
# VIP : 192.168.80.10
# ============================================================

MASTERS = [
  { name: "masters1", ip: "192.168.80.11" },
  { name: "masters2", ip: "192.168.80.12" },
  { name: "masters3", ip: "192.168.80.13" },
]

WORKERS = [
  { name: "workers1", ip: "192.168.80.21" },
  { name: "workers2", ip: "192.168.80.22" },
]

VIP          = "192.168.80.10"
# IFACE        = "eth1"          # interface host-only VirtualBox
IFACE        = "enp0s8"          # interface host-only VirtualBox
VRRP_ID      = 51
BOX          = "ubuntu/jammy64"
MASTER_CPU   = 2
MASTER_MEM   = 4096
WORKER_CPU   = 2
WORKER_MEM   = 3072

Vagrant.configure("2") do |config|

  config.vm.box = BOX

  # ── Masters ─────────────────────────────────────────────
  MASTERS.each_with_index do |m, idx|
    priority = 110 - (idx * 10)   # master1=110, master2=100, master3=90

    config.vm.define m[:name] do |node|
      node.vm.hostname = m[:name]
      node.vm.network "private_network", ip: m[:ip],
        virtualbox__intnet: false,
        nic_type: "82540EM"

      node.vm.provider "virtualbox" do |vb|
        vb.name   = m[:name]
        vb.cpus   = MASTER_CPU
        vb.memory = MASTER_MEM
        # Requis pour VRRP multicast Keepalived
        vb.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
      end

      node.vm.provision "shell", path: "common_script.sh",
        args: ["master", m[:ip], VIP, IFACE, VRRP_ID.to_s, priority.to_s,
               MASTERS.map { |x| x[:ip] }.join(",")]
    end
  end

  # ── Workers ─────────────────────────────────────────────
  WORKERS.each do |w|
    config.vm.define w[:name] do |node|
      node.vm.hostname = w[:name]
      node.vm.network "private_network", ip: w[:ip],
        nic_type: "82540EM"

      node.vm.provider "virtualbox" do |vb|
        vb.name   = w[:name]
        vb.cpus   = WORKER_CPU
        vb.memory = WORKER_MEM
      end

      node.vm.provision "shell", path: "common_script.sh",
        args: ["worker", w[:ip], VIP]
    end
  end

end
