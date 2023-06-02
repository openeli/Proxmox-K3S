# Proxmox-K3S
How to setup a Kubernetes Cluster on Proxmox running on Raspberry pi 4

Prerequisites

To follow along with this tutorial, you’ll need:

At least two Ubuntu Server 22.04 instances (one for the controller, one for at least one node)
A static IP address or DHCP reservation on those instances, so their IP addresses cannot change
The controller node should have at least 2GB of RAM and 2 cores
The node instances can have a single gigabyte of RAM and a single CPU core (or more if you prefer)
Note: This tutorial utilizes Ubuntu Server 22.04. If you choose to use a different distribution or a different version of Ubuntu, these commands may not work.

Setting up the cluster

The following commands are to be run on the controller node (unless otherwise stated).

Setting up a static IP

Note, a static lease is preferred, but you can use the following Netplan config example to serve as a basis of yours if you need to set up a static IP.

network:
    version: 2
    ethernets:
        eth0:
            addresses: [10.10.10.213/24]
            nameservers:
                addresses: [10.10.10.1]
            routes:
                - to: default
                  via: 10.10.10.1
Installing containerd

This Kubernetes cluster will utilize the containerd runtime. To set that up, we’ll first need to install the required package:

sudo apt install containerd
Create the initial configuration:

sudo mkdir /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
For this cluster to work properly, we’ll need to enable SystemdCgroup within the configuration. To do that, we’ll need to edit the config we’ve just created:

sudo nano /etc/containerd/config.toml
Within that file, find the following line of text:

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
Underneath that, find the SystemdCgroup option and change it to true, which should look like this:

SystemdCgroup = true
Disable swap

Kubernetes requires swap to be disabled. To turn off swap, we can run the following command:

sudo swapoff -a
Next, edit the /etc/fstab file and comment out the line that corresponds to swap (if present). This will help ensure swap doesn’t end up getting turned back on after reboot.

Enable bridging

To enable bridging, we only need to edit one config file:

sudo nano /etc/sysctl.conf
Within that file, look for the following line:

#net.ipv4.ip_forward=1
Uncomment that line by removing the # symbol in front of it, which should make it look like this:

net.ipv4.ip_forward=1
Enable br_netfilter

The next step is to enable br_netfilter by editing yet another config file:

sudo nano /etc/modules-load.d/k8s.conf
Add the following to that file (the file should actually be empty at first):

br_netfilter
Reboot your servers

Reboot each of your instances to ensure all of our changes so far are in place:

sudo reboot



# Preparing the Node

Preparing the Node to recive new ID at cloning 

sudo cloud-init clean
sudo rm -rf /var/lib/cloud/instances
sudo truncate -s 0 /etc/machine-id
sudo rm /var/lib/dbus/machine-id
sudo ln -s /etc/machine-id /var/lib/dbus/machine-id

Cheking if it is linked
ls -l /var/lib/dbus/machine-id

The output must be empty
cat /etc/machine-id

sudo shutdown -h now


# Installing Kubernetes

You can force the OS to use legacy iptables to ensure compatibility with older Kubernetes versions, but this step is optional.

iptables -F
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
Install and test k3s.

curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -

# After a minute, you should be able to test it
kubectl get nodes
The kubeconfig yaml will be available at /etc/rancher/k3s/k3s.yaml.

If you have worker nodes to add, use the following command to do so. The cluster token can be found at /var/lib/rancher/k3s/server/node-token on the server node.

curl -sfL https://get.k3s.io | K3S_URL=https://<server_ip>:6443 K3S_TOKEN="<cluster_token>" sh -
Maintenance

Things should run smoothly from here, but, if they don’t, the next few commands might help:

# See current status of k3s
systemctl status k3s

# See system logs (including k3s)
journalctl -xe

# Uninstall k3s
/usr/local/bin/k3s-uninstall.sh
Kubernetes automatically performs cleanup of unused containers and images on nodes, but if you’re under the impression that your filesystem is getting bloated by these, you can force a safe deletion with:

k3s crictl rmi --prune

Sources:
https://www.learnlinux.tv/how-to-build-an-awesome-kubernetes-cluster-using-proxmox-virtual-environment/

https://laury.dev/snippets/install-kubernetes-raspberry-pi/
