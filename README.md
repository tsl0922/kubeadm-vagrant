# kubeadm-vagrant

Run kubernetes cluster with kubeadm on vagrant.

> **Reference:** [Creating Highly Available clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

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

# Pod Network

- Reference: [Installing a Pod network add-on](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)
- `kube-flannel.yml` changes: added the `--iface` option ([ref](https://github.com/coreos/flannel/blob/master/Documentation/troubleshooting.md#vagrant))


# Credits

- based on: https://github.com/coolsvap/kubeadm-vagrant