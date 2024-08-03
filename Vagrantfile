Vagrant.require_version ">= 2.0.0"

boxes = [
    {
        :name => "kube-control-plane",
        :eth1 => "192.168.56.10",
        :mem => "3072",
        :cpu => "2"
    },
    {
        :name => "kube-node1",
        :eth1 => "192.168.56.11",
        :mem => "1024",
        :cpu => "1"
#    },
#    {
#        :name => "kube-node2",
#        :eth1 => "192.168.56.12",
#        :mem => "1024",
#        :cpu => "1"
    }
]

Vagrant.configure(2) do |config|
  config.vm.box = "generic/ubuntu2204"

  config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")

  boxes.each do |opts|
      config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.provider "virtualbox" do |v|
          v.customize ["modifyvm", :id, "--memory", opts[:mem]]
          v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
          v.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          v.customize ["modifyvm", :id, "--uartmode1", "file", File::NULL]
        end
        config.vm.network :private_network, ip: opts[:eth1]
      end
  end

  config.vm.provision "shell", inline: <<-SHELL
    sudo modprobe overlay
    sudo modprobe br_netfilter
    sudo echo 'overlay' > /etc/modules-load.d/containerd.conf
    sudo echo 'br_netfilter' >> /etc/modules-load.d/containerd.conf
    sudo echo 'net.bridge.bridge-nf-call-iptables = 1' > /etc/sysctl.d/kubernetes.conf
    sudo echo 'net.bridge.bridge-nf-call-ip6tables = 1' >> /etc/sysctl.d/kubernetes.conf
    sudo echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.d/kubernetes.conf
    sudo sysctl --system

    sudo apt update
    sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

#    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
#    echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> ~/kubernetes.list
#    sudo mv ~/kubernetes.list /etc/apt/sources.list.d
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    sudo apt update
    echo "KUBELET_EXTRA_ARGS=--node-ip="$(ip addr show eth1  | awk '$1 == "inet" { print $2 }' | cut -d/ -f1) | sudo tee /etc/default/kubelet

    sudo apt install -y containerd.io
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
    sudo systemctl restart containerd

    # Install Docker
    sudo snap install docker 

    # Install etcdctl
    export RELEASE=$(curl -s https://api.github.com/repos/etcd-io/etcd/releases/latest|grep tag_name | cut -d '"' -f 4)
    wget https://github.com/etcd-io/etcd/releases/download/${RELEASE}/etcd-${RELEASE}-linux-amd64.tar.gz
    tar xvf etcd-${RELEASE}-linux-amd64.tar.gz
    cd etcd-${RELEASE}-linux-amd64
    sudo mv etcd etcdctl etcdutl /usr/local/bin 

    sudo apt install -y kubelet kubeadm kubectl

    sudo swapoff -a
    sudo sed -i '/ swap / s/^/#/' /etc/fstab

    sudo crictl config runtime-endpoint unix:///run/containerd/containerd.sock
    
    # Install Jenkins
    sudo apt install -y openjdk-17-jdk
    sudo wget -O /usr/share/keyrings/jenkins-keyring.asc  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
    echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]"  https://pkg.jenkins.io/debian-stable binary/ | sudo tee  /etc/apt/sources.list.d/jenkins.list > /dev/null
    sudo apt-get update
    sudo apt-get install -y jenkins

  SHELL

end
