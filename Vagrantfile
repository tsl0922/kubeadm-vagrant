BOX_IMAGE = "ubuntu/xenial64"
MASTER_COUNT = 1
NODE_COUNT = 2
MASTER_IP = "192.168.26.10"
MASTER_PORT = "8443"
NODE_IP_NW = "192.168.26."
POD_NW_CIDR = "10.244.0.0/16"

DOCKER_VER = "18.06.2~ce~3-0~ubuntu"
KUBE_VER = "1.14.1"
KUBE_TOKEN = "ayngk7.m1555duk5x2i3ctt"
IMAGE_REPO = "registry.aliyuncs.com/google_containers"

def gen_haproxy_backend(master_count)
  server=""
  (1..master_count).each do |i|
    ip = NODE_IP_NW + "#{i + 10}"
    server << "    server apiserver#{i} #{ip}:6443 check\n"
  end
  server
end

init_script = <<SCRIPT
#!/bin/bash

set -eo pipefail

cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_nonlocal_bind = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sysctl --system

echo "apt_preserve_sources_list: true" > /etc/cloud/cloud.cfg.d/99-apt-preserve-sources-list.cfg
sed -i 's/\\(archive\\|security\\)\\.ubuntu\\.com/mirrors\\.aliyun\\.com/g' /etc/apt/sources.list
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
cat > /etc/apt/sources.list.d/docker-ce.list <<EOF
deb http://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial stable
EOF
apt-get update && apt-get install -y docker-ce=#{DOCKER_VER}

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update && apt-get install -y kubelet=#{KUBE_VER}-00 kubeadm kubectl kubernetes-cni=0.7.5-00

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

set -eo pipefail

discovery_token_ca_cert_hash="$(grep 'discovery-token-ca-cert-hash' /vagrant/kubeadm.log | head -n1 | awk '{print $2}')"
kubeadm reset -f
kubeadm join #{MASTER_IP}:#{MASTER_PORT} --token #{KUBE_TOKEN} --discovery-token-ca-cert-hash ${discovery_token_ca_cert_hash}
SCRIPT

master_script = <<SCRIPT
#!/bin/bash

set -eo pipefail

kubeadm reset -f
kubeadm init --image-repository #{IMAGE_REPO} \
             --apiserver-advertise-address=#{MASTER_IP} \
             --apiserver-bind-port=#{MASTER_PORT} \
             --kubernetes-version v#{KUBE_VER} \
             --pod-network-cidr=#{POD_NW_CIDR} \
             --token #{KUBE_TOKEN} \
             --token-ttl 0 | tee /vagrant/kubeadm.log

mkdir -p $HOME/.kube
sudo cp -Rf /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
SCRIPT

ha_script = <<SCRIPT
#!/bin/bash

set -eo pipefail

status() {
    echo -e "\033[35m >>>   $*\033[0;39m"
}

status "configuring haproxy and keepalived.."
apt-get install -y keepalived haproxy

systemctl stop keepalived || true

vrrp_if=$(ip a | grep 192.168.26 | awk '{print $7}')
vrrp_ip=$(ip a | grep 192.168.26 | awk '{split($2, a, "/"); print a[1]}')
vrrp_state="BACKUP"
vrrp_priority="100"
if [ "${vrrp_ip}" = "#{NODE_IP_NW}11" ]; then
  vrrp_state="MASTER"
  vrrp_priority="101"
fi

cat > /etc/keepalived/keepalived.conf <<EOF
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}
vrrp_instance VI_1 {
    state ${vrrp_state}
    interface ${vrrp_if}
    mcast_src_ip ${vrrp_ip}
    virtual_router_id 51
    priority ${vrrp_priority}
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass a6E/CHhJkCn1Ww1gF3qPiJTKTEc=
    }
    virtual_ipaddress {
        #{MASTER_IP}
    }
    track_script {
       check_apiserver
    }
}
EOF

cat > /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/bash

errorExit() {
  echo "*** $*" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q #{MASTER_IP}; then
  curl --silent --max-time 2 --insecure https://#{MASTER_IP}:#{MASTER_PORT}/ -o /dev/null || errorExit "Error GET https://#{MASTER_IP}:#{MASTER_PORT}/"
fi
EOF

systemctl restart keepalived
sleep 10

cat > /etc/haproxy/haproxy.cfg <<EOF
global
  log /dev/log  local0
  log /dev/log  local1 notice
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5s
    timeout client 50s
    timeout client-fin 50s
    timeout server 50s
    timeout tunnel 1h

listen stats
    bind *:1080
    stats refresh 30s
    stats uri /stats

listen kube-api-server
    bind #{MASTER_IP}:#{MASTER_PORT}
    mode tcp
    option tcplog
    balance roundrobin

#{gen_haproxy_backend(MASTER_COUNT)}
EOF

systemctl restart haproxy

if [ ${vrrp_state} = "MASTER" ]; then
  cat > /tmp/kubeadm-config.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta1
kind: InitConfiguration
bootstrapTokens:
- token: #{KUBE_TOKEN}
  ttl: 24h
localAPIEndpoint:
  advertiseAddress: ${vrrp_ip}
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v#{KUBE_VER}
controlPlaneEndpoint: "#{MASTER_IP}:#{MASTER_PORT}"
imageRepository: #{IMAGE_REPO}
networking:
  podSubnet: 10.244.0.0/16
EOF

  status "running kubeadm init on the first master node.."
  kubeadm reset -f
  kubeadm init --config=/tmp/kubeadm-config.yaml --experimental-upload-certs | tee /vagrant/kubeadm.log

  mkdir -p $HOME/.kube
  sudo cp -Rf /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
  status "installing flannel network addon.."
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
else
  status "joining master node.."
  discovery_token_ca_cert_hash="$(grep 'discovery-token-ca-cert-hash' /vagrant/kubeadm.log | head -n1 | awk '{print $2}')"
  certificate_key="$(grep 'certificate-key' /vagrant/kubeadm.log | head -n1 | awk '{print $3}')"
  kubeadm reset -f
  kubeadm join #{MASTER_IP}:#{MASTER_PORT} --token #{KUBE_TOKEN} \
    --discovery-token-ca-cert-hash ${discovery_token_ca_cert_hash} \
    --experimental-control-plane --certificate-key ${certificate_key} \
    --apiserver-advertise-address ${vrrp_ip}
fi
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

  (1..MASTER_COUNT).each do |i|
    ha = MASTER_COUNT > 1
    hostname= "master#{ha ? i: ''}"
    config.vm.define(hostname) do |subconfig|
      subconfig.vm.hostname = hostname
      subconfig.vm.network :private_network, nic_type: "virtio", ip: ha ? NODE_IP_NW + "#{i + 10}" : MASTER_IP
      subconfig.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--cpus", "2"]
        vb.customize ["modifyvm", :id, "--memory", "2048"]
      end
      subconfig.vm.provision :shell, inline: ha ? ha_script : master_script
    end
  end

  (1..NODE_COUNT).each do |i|
    config.vm.define("node#{i}") do |subconfig|
      subconfig.vm.hostname = "node#{i}"
      subconfig.vm.network :private_network, nic_type: "virtio", ip: NODE_IP_NW + "#{i + 20}"
      subconfig.vm.provision :shell, inline: node_script
    end
  end
end
