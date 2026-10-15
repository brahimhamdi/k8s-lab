# ============================================================
# Kubernetes Cluster - Vagrant + VirtualBox
# ============================================================
#
# OS          : Ubuntu 24.04 LTS (Noble)
# Kubernetes  : v1.36
# Runtime     : containerd
# CRI CLI     : crictl
#
# Architecture :
#
#   kube-control-plane : 192.168.56.10 - 4 Go RAM - 2 CPU
#   kube-node1         : 192.168.56.11 - 2 Go RAM - 2 CPU
#   kube-node2         : 192.168.56.12 - 2 Go RAM - 2 CPU
#
# ============================================================

Vagrant.require_version ">= 2.0.0"

# ============================================================
# 1. Configuration générale
# ============================================================
KUBERNETES_VERSION = "v1.36"
BOX_NAME = "ubuntu64/noble64"
boxes = [
  {
    name: "kube-control-plane",
    ip: "192.168.56.10",
    mem: 4096,
    cpu: 2
  },
  {
    name: "kube-node1",
    ip: "192.168.56.11",
    mem: 2048,
    cpu: 2
  },
  {
    name: "kube-node2",
    ip: "192.168.56.12",
    mem: 2048,
    cpu: 2
  }
]

# ============================================================
# 2. Configuration Vagrant
# ============================================================
Vagrant.configure("2") do |config|
  config.vm.box = BOX_NAME
  # Désactiver la mise à jour automatique des Guest Additions
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # Désactiver le dossier partagé par défaut
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # ==========================================================
  # 3. Création des machines virtuelles
  # ==========================================================
  boxes.each do |opts|
    config.vm.define opts[:name] do |node|
      node.vm.hostname = opts[:name]

      # Configuration réseau
      node.vm.network "private_network", ip: opts[:ip]

      # Configuration VirtualBox
      node.vm.provider "virtualbox" do |vb|

        vb.name = opts[:name]

        vb.memory = opts[:mem]
        vb.cpus = opts[:cpu]

        vb.customize [
          "modifyvm", :id,
          "--uart1", "0x3F8", "4"
        ]

        vb.customize [
          "modifyvm", :id,
          "--uartmode1", "file", File::NULL
        ]

      end

      # ======================================================
      # 4. Provisionnement commun à tous les nœuds
      # ======================================================
      node.vm.provision "shell",
        name: "Install Kubernetes prerequisites",
        privileged: true,
        env: {
          "K8S_VERSION" => KUBERNETES_VERSION
        },
        inline: <<-'SHELL'

#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

echo "=========================================="
echo "Starting Kubernetes node provisioning"
echo "=========================================="

# ----------------------------------------------------------
# 4.1 Mise à jour du système
# ----------------------------------------------------------
apt-get update
apt-get install -y curl wget gnupg ca-certificates apt-transport-https software-properties-common bash-completion binutils tree vim git jq unzip tar conntrack socat

# ----------------------------------------------------------
# 4.2 Configuration Bash pour l'utilisateur vagrant
# ----------------------------------------------------------

BASHRC="/home/vagrant/.bashrc"
cat >> "$BASHRC" <<'EOF'

# Kubernetes aliases
alias k=kubectl
alias c=clear

# Vim configuration
export EDITOR=vim

# Kubernetes completion
if command -v kubectl >/dev/null 2>&1; then
    source <(kubectl completion bash)
    complete -F __start_kubectl k
fi

EOF

cat > /home/vagrant/.vimrc <<'EOF'
colorscheme ron
set tabstop=2
set shiftwidth=2
set expandtab
set number
set autoindent
EOF

chown vagrant:vagrant /home/vagrant/.bashrc /home/vagrant/.vimrc

# ----------------------------------------------------------
# 4.3 Désactivation du swap
# ----------------------------------------------------------
echo "Disabling swap..."
swapoff -a

# Désactiver les entrées swap dans /etc/fstab
sed -i.bak -E '/^[[:space:]]*[^#].*[[:space:]]swap[[:space:]]/s/^/#/' /etc/fstab

# Désactiver les éventuelles unités swap systemd
systemctl daemon-reload

# ----------------------------------------------------------
# 4.4 Chargement des modules du noyau
# ----------------------------------------------------------

echo "Configuring kernel modules..."

cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

# ----------------------------------------------------------
# 4.5 Configuration réseau Kubernetes
# ----------------------------------------------------------
echo "Configuring sysctl..."

cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sysctl --system

# ----------------------------------------------------------
# 4.6 Installation de containerd
# ----------------------------------------------------------
echo "Installing containerd..."
install -m 0755 -d /etc/apt/keyrings

# Clé officielle Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Dépôt Docker officiel
. /etc/os-release
ARCH=$(dpkg --print-architecture)
cat > /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: ${VERSION_CODENAME}
Components: stable
Architectures: ${ARCH}
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt-get update
apt-get install -y containerd.io

# ----------------------------------------------------------
# 4.7 Configuration de containerd
# ----------------------------------------------------------
echo "Configuring containerd..."
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# Utiliser systemd comme gestionnaire de cgroups
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

# Vérifier que le plugin CRI n'est pas désactivé
if grep -Eq 'disabled_plugins[[:space:]]*=[[:space:]]*\[[^]]*"cri"' /etc/containerd/config.toml; then
    echo "ERROR: containerd CRI plugin is disabled"
    exit 1
fi

# Activer containerd
systemctl enable containerd
systemctl restart containerd

# Vérifier le service
systemctl is-active --quiet containerd

# ----------------------------------------------------------
# 4.8 Installation du dépôt Kubernetes
# ----------------------------------------------------------
echo "Configuring Kubernetes repository..."
mkdir -p -m 0755 /etc/apt/keyrings
curl -fsSL "https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/Release.key" | gpg --batch --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_VERSION}/deb/ /
EOF

apt-get update

# ----------------------------------------------------------
# 4.9 Installation de Kubernetes
# ----------------------------------------------------------
echo "Installing Kubernetes components..."
apt-get install -y kubelet kubeadm kubectl

# ----------------------------------------------------------
# 4.10 Installation de crictl
# ----------------------------------------------------------
echo "Installing cri-tools..."

# Installation depuis le dépôt Kubernetes
apt-get install -y cri-tools

# Vérification
command -v crictl
crictl --version

# ----------------------------------------------------------
# 4.11 Configuration de crictl
# ----------------------------------------------------------
echo "Configuring crictl..."
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

chmod 644 /etc/crictl.yaml

# ----------------------------------------------------------
# 4.12 Configuration de kubelet
# ----------------------------------------------------------
systemctl enable kubelet

# kubelet peut rester inactif avant kubeadm init/join
systemctl start kubelet || true

# ----------------------------------------------------------
# 4.13 Bloquer les mises à jour automatiques Kubernetes
# ----------------------------------------------------------
apt-mark hold kubelet kubeadm kubectl cri-tools

# ----------------------------------------------------------
# 4.14 Configuration des noms des nœuds
# ----------------------------------------------------------
echo "Configuring /etc/hosts..."
sed -i '/# BEGIN K8S LAB/,/# END K8S LAB/d' /etc/hosts
cat >> /etc/hosts <<EOF

# BEGIN K8S LAB
192.168.56.10 kube-control-plane
192.168.56.11 kube-node1
192.168.56.12 kube-node2
# END K8S LAB

EOF

# ----------------------------------------------------------
# 4.15 Vérifications
# ----------------------------------------------------------
echo "=========================================="
echo "Checking installed components"
echo "=========================================="
containerd --version
crictl --version
kubeadm version -o short
kubectl version --client
systemctl is-active --quiet containerd
crictl info > /dev/null

echo "=========================================="
echo "Node provisioning completed successfully"
echo "=========================================="

SHELL

      # ======================================================
      # 5. Provisionnement du control-plane uniquement
      # ======================================================

      if opts[:name] == "kube-control-plane"

        node.vm.provision "shell",
          name: "Install control-plane tools",
          privileged: true,
          inline: <<-'SHELL'

#!/usr/bin/env bash
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive
echo "=========================================="
echo "Installing control-plane tools"
echo "=========================================="

# ----------------------------------------------------------
# 5.1 Installation de Docker
# ----------------------------------------------------------
echo "Installing Docker..."
apt-get update
apt-get install -y docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin
systemctl enable docker
systemctl start docker

# Autoriser l'utilisateur vagrant à utiliser Docker
usermod -aG docker vagrant

# ----------------------------------------------------------
# 5.2 Installation des outils de développement
# ----------------------------------------------------------
echo "Installing development tools..."
apt-get install -y git openjdk-21-jdk maven mariadb-client jq wget curl unzip

# ----------------------------------------------------------
# 5.3 Installation de Helm
# ----------------------------------------------------------
echo "Installing Helm..."
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 -o /tmp/get_helm.sh
chmod 700 /tmp/get_helm.sh
/tmp/get_helm.sh
rm -f /tmp/get_helm.sh

# Configuration de l'autocomplétion Helm
cat >> /home/vagrant/.bashrc <<'EOF'

# Helm completion
if command -v helm >/dev/null 2>&1; then
    source <(helm completion bash)
fi

EOF

# ----------------------------------------------------------
# 5.4 Installation de Kustomize
# ----------------------------------------------------------
echo "Installing Kustomize..."
curl -fsSL https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh -o /tmp/install_kustomize.sh
chmod +x /tmp/install_kustomize.sh
cd /tmp
./install_kustomize.sh
install -m 0755 kustomize /usr/local/bin/kustomize

rm -f /tmp/kustomize /tmp/install_kustomize.sh

# ----------------------------------------------------------
# 5.5 Installation de etcdctl et etcdutl
# ----------------------------------------------------------
echo "Installing etcd tools..."

# Version stable choisie pour les outils du laboratoire
ETCD_VERSION="v3.6.5"
ETCD_ARCH="amd64"
ETCD_FILE="etcd-${ETCD_VERSION}-linux-${ETCD_ARCH}.tar.gz"

cd /tmp
curl -fL "https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/${ETCD_FILE}" -o "${ETCD_FILE}"
tar -xzf "${ETCD_FILE}"
ETCD_DIR="etcd-${ETCD_VERSION}-linux-${ETCD_ARCH}"
install -m 0755 "${ETCD_DIR}/etcdctl" /usr/local/bin/etcdctl
install -m 0755 "${ETCD_DIR}/etcdutl" /usr/local/bin/etcdutl
rm -rf \
  "${ETCD_DIR}" \
  "${ETCD_FILE}"

# ----------------------------------------------------------
# 5.6 Configuration finale
# ----------------------------------------------------------
chown vagrant:vagrant /home/vagrant/.bashrc

# ----------------------------------------------------------
# 5.7 Vérification des outils
# ----------------------------------------------------------
echo "=========================================="
echo "Checking control-plane tools"
echo "=========================================="
docker --version
helm version --short
kustomize version
etcdctl version
java -version
mvn -version
git --version
echo "=========================================="
echo "Control-plane provisioning completed"
echo "=========================================="

SHELL

      end

    end
  end
end
