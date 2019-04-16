BOX_IMAGE = "ubuntu/xenial64"
NODE_COUNT = 2
MASTER_IP = "192.168.26.10"
NODE_IP_NW = "192.168.26."
POD_NW_CIDR = "10.244.0.0/16"

DOCKER_VER = "18.06.2~ce~3-0~ubuntu"
KUBE_VER = "1.13.5"
KUBE_TOKEN = "ayngk7.m1555duk5x2i3ctt"
IMAGE_REPO = "registry.aliyuncs.com/google_containers"

init_script = <<SCRIPT
#!/bin/bash

sed -i 's/\\(archive\\|security\\)\\.ubuntu\\.com/mirrors\\.aliyun\\.com/g' /etc/apt/sources.list
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
cat > /etc/apt/sources.list.d/docker-ce.list <<EOF
deb http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial stable
EOF
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y docker-ce=#{DOCKER_VER} kubelet=#{KUBE_VER}-00 kubeadm=#{KUBE_VER}-00 kubectl=#{KUBE_VER}-00 kubernetes-cni=0.7.5-00

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d

# https://github.com/kubernetes/kubernetes/issues/45487
echo 'User=root' >> /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

systemctl daemon-reload
systemctl enable kubelet && systemctl restart kubelet
systemctl enable docker && systemctl restart docker
usermod -aG docker vagrant
SCRIPT

node_script = <<SCRIPT
#!/bin/bash

kubeadm reset -f
kubeadm join --discovery-token-unsafe-skip-ca-verification --token #{KUBE_TOKEN} #{MASTER_IP}:6443
SCRIPT

master_script = <<SCRIPT
#!/bin/bash

kubeadm reset -f
kubeadm init --image-repository #{IMAGE_REPO} \
             --apiserver-advertise-address=#{MASTER_IP} \
             --kubernetes-version v#{KUBE_VER} \
             --pod-network-cidr=#{POD_NW_CIDR} \
             --token #{KUBE_TOKEN} \
             --token-ttl 0

mkdir -p $HOME/.kube
sudo cp -Rf /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
SCRIPT

Vagrant.configure("2") do |config|
  config.vm.box = BOX_IMAGE
  config.vm.box_check_update = false

  config.vm.provider "virtualbox" do |l|
    l.cpus = 1
    l.memory = "1024"
  end

  config.vm.provision :shell, inline: init_script

  config.hostmanager.enabled = true
  config.hostmanager.manage_guest = true

  config.vm.define "master" do |subconfig|
    subconfig.vm.hostname = "master"
    subconfig.vm.network :private_network, nic_type: "virtio", ip: MASTER_IP
    subconfig.vm.provider :virtualbox do |vb|
      vb.customize ["modifyvm", :id, "--cpus", "2"]
      vb.customize ["modifyvm", :id, "--memory", "2048"]
    end
    subconfig.vm.provision :shell, inline: master_script
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define "node#{i}" do |subconfig|
      subconfig.vm.hostname = "node#{i}"
      subconfig.vm.network :private_network, nic_type: "virtio", ip: NODE_IP_NW + "#{i + 10}"
      subconfig.vm.provision :shell, inline: node_script
    end
  end
end
