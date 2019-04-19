# kubeadm-vagrant

Run kubernetes cluster with kubeadm on vagrant.

# Requirements

1. virtualbox: https://www.virtualbox.org/wiki/Downloads
2. vagrant: https://www.vagrantup.com/downloads.html
3. `vagrant plugin install vagrant-hostmanager`

# Usage

Run `vagrant up` and wait for the cluster to be set up (change `MASTER_COUNT` to `3` to run a ha cluster).

To use `kubectl` on the master node, run:

```bash
vagrant ssh master # use master1 if you are running ha cluster

mkdir -p $HOME/.kube
sudo cp -Rf /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl cluster-info
kubectl get nodes
```

# Credits

- based on: https://github.com/coolsvap/kubeadm-vagrant