# -*- mode: ruby -*-
# vi:set ft=ruby sw=2 ts=2 sts=2:

NUM_MASTER_NODE = 1
NUM_WORKER_NODE = 2

IP_NW = "192.168.56."
MASTER_IP_START = 1
NODE_IP_START = 2

BOX = "ubuntu/jammy64"   # upgraded from bionic for modern k8s

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.box_check_update = false

  # Common provisioning for ALL nodes
  common_bootstrap = <<-SHELL
    set -euxo pipefail
    export DEBIAN_FRONTEND=noninteractive

    # Basic tooling
    apt-get update -y
    apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

    # Disable swap (required for kubeadm’s default path)
    swapoff -a
    sed -ri 's/(^\\S+\\s+\\S+\\s+swap\\s+\\S+\\s+\\S+\\s+\\S+)/# \\1/' /etc/fstab

    # Kernel modules + sysctls for container networking
    cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
    modprobe overlay
    modprobe br_netfilter
    cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
    sysctl --system

    # Install and configure containerd (systemd cgroups)
    apt-get install -y containerd
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml > /dev/null
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    systemctl enable --now containerd

    # Kubernetes apt repo (v1.33 stable) + tools
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
      | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" \
      | tee /etc/apt/sources.list.d/kubernetes.list
    apt-get update -y
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    systemctl enable kubelet

    # /etc/hosts convenience
    grep -q kubemaster /etc/hosts || cat <<EOT >> /etc/hosts
#{IP_NW}#{MASTER_IP_START + 1} kubemaster
#{IP_NW}#{NODE_IP_START + 1} kubenode01
#{IP_NW}#{NODE_IP_START + 2} kubenode02
EOT
  SHELL

  # Master
  (1..NUM_MASTER_NODE).each do |i|
    config.vm.define "kubemaster" do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "kubemaster"
        vb.memory = 3072     # 2GB works; 3GB is smoother for the control plane
        vb.cpus = 2
      end
      node.vm.hostname = "kubemaster"
      node.vm.network :private_network, ip: IP_NW + "#{MASTER_IP_START + i}"  # 192.168.56.2
      node.vm.network "forwarded_port", guest: 22, host: "#{2710 + i}"
      node.vm.provision "shell", inline: common_bootstrap
    end
  end

  # Workers
  (1..NUM_WORKER_NODE).each do |i|
    config.vm.define "kubenode0#{i}" do |node|
      node.vm.provider "virtualbox" do |vb|
        vb.name = "kubenode0#{i}"
        vb.memory = 2048
        vb.cpus = 2
      end
      node.vm.hostname = "kubenode0#{i}"
      node.vm.network :private_network, ip: IP_NW + "#{NODE_IP_START + i}"  # .3 and .4
      node.vm.network "forwarded_port", guest: 22, host: "#{2720 + i}"
      node.vm.provision "shell", inline: common_bootstrap
    end
  end
end
