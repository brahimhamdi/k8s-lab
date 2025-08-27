Vagrant.require_version ">= 2.0.0"

boxes = [
  {
      :name => "kube-control-plane",
      :eth1 => "192.168.56.10",
      :mem => "6144",
      :cpu => "4"
  },
  {
      :name => "kube-node1",
      :eth1 => "192.168.56.11",
      :mem => "2048",
      :cpu => "1"
  },
  {
      :name => "kube-node2",
      :eth1 => "192.168.56.12",
      :mem => "2048",
      :cpu => "1"
  }
]

Vagrant.configure(2) do |config|
  config.vm.box = "generic/ubuntu2204"

  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

  boxes.each do |opts|
    config.vm.define opts[:name] do |node|
      node.vm.hostname = opts[:name]

      node.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
        v.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
        v.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
      end

      node.vm.network :private_network, ip: opts[:eth1]

      node.vm.provision "shell", inline: <<-SHELL
      # provision all nodes
        # setup terminal ############################################################
        apt --allow-unauthenticated update
        apt --allow-unauthenticated install -y bash-completion binutils
        echo 'colorscheme ron' >> ~/.vimrc
        echo 'set tabstop=2' >> ~/.vimrc
        echo 'set shiftwidth=2' >> ~/.vimrc
        echo 'set expandtab' >> ~/.vimrc
        echo 'source <(kubectl completion bash)' >> ~/.bashrc
        echo 'alias k=kubectl' >> ~/.bashrc
        echo 'alias c=clear' >> ~/.bashrc
        echo 'complete -F __start_kubectl k' >> ~/.bashrc
        sed -i '1s/^/force_color_prompt=yes\n/' ~/.bashrc

        # modules & fonctionnalities #################################################
        sudo modprobe overlay
        sudo modprobe br_netfilter
        echo 'overlay' | sudo tee /etc/modules-load.d/containerd.conf
        echo 'br_netfilter' | sudo tee -a /etc/modules-load.d/containerd.conf
        echo 'net.bridge.bridge-nf-call-iptables = 1' | sudo tee /etc/sysctl.d/kubernetes.conf
        echo 'net.bridge.bridge-nf-call-ip6tables = 1' | sudo tee -a /etc/sysctl.d/kubernetes.conf
        echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/kubernetes.conf
        sudo sysctl --system

        # Install & config containerd #################################################
        sudo apt update
        sudo apt install -y apt-transport-https ca-certificates curl software-properties-common tree
        curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
        sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

        sudo apt install -y containerd.io
        sudo containerd config default | sudo tee /etc/containerd/config.toml
        sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
        sudo systemctl restart containerd
        sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock

        # Install kubernetes ##############################################################
        sudo mkdir -p /etc/apt/keyrings
        echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
        curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
        sudo apt update

        # Install etcdctl
        RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
        wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
        tar xvf etcd-${RELEASE}-linux-amd64.tar.gz
        cd etcd-${RELEASE}-linux-amd64
        sudo mv etcd etcdctl etcdutl /usr/local/bin
        cd ..

        sudo apt install -y kubelet kubeadm

        # swap off
        sudo swapoff -a

      SHELL

      # Provision only control-plane (Helm + Kustomize + docker + openjdk + maven + ...)
      if opts[:name] == "kube-control-plane"
        node.vm.provision "shell", inline: <<-SHELL

          # Install Kubectl
          sudo apt install -y kubectl
          echo "source <(kubectl completion bash)" >> ~/.bashrc

          # Install Docker, openjdk, maven, ...
          sudo apt install -y git docker-ce openjdk-17-jdk maven 
          sudo usermod -aG docker vagrant

          # Install Helm
          sudo snap install helm --classic
          echo "source <(helm completion bash)" >> ~/.bashrc

          # Install Kustomize
          wget "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"
          chmod +x install_kustomize.sh
          ./install_kustomize.sh
          sudo mv kustomize /usr/bin/
        SHELL
      end
    end
  end
end
