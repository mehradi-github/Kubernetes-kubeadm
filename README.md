# Kubernetes Cluster installation using kubeadm
[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool built to provide kubeadm init and kubeadm join as best-practice "fast paths" for creating Kubernetes clusters.

kubeadm performs the actions necessary to get a minimum viable cluster up and running. By design, it cares only about bootstrapping, not about provisioning machines.

<!-- TABLE OF CONTENTS -->
## Table of Contents
- [Kubernetes Cluster installation using kubeadm](#kubernetes-cluster-installation-using-kubeadm)
  - [Table of Contents](#table-of-contents)
  - [Installing kubeadm](#installing-kubeadm)
    - [System Requirements](#system-requirements)
    - [Control plane](#control-plane)
    - [Worker node(s)](#worker-nodes)

## Installing kubeadm
This page shows how to [install the kubeadm toolbox](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) on Amazon Linux 2 as a virtual machine.

### System Requirements
- A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps).
- 2 CPUs or more.
- Full network connectivity between all machines in the cluster (public or private network is fine).
- Unique hostname, MAC address, and product_uuid for every node.
- Swap disabled. You MUST disable swap in order for the kubelet to work properly.

```sh
ip link 
ifconfig -a
sudo cat /sys/class/dmi/id/product_uuid

hostnamectl set-hostname 'gfs-server-03'
hostnamectl

sudo swapoff -a

# Check required ports
sudo yum install nc
sudo yum install iptables-services

sudo systemctl enable iptables
sudo systemctl start iptables
sudo systemctl status iptables

#List all open ports
# This will print all listening sockets (-l) along with the port number (-n), with TCP ports (-t) and UDP ports (-u) also listed in the output.
netstat -lntu

#list listening sockets with an open port
ss -lntu

# This sets the firewall to append (-A) the new rule to accept input packets via protocol (-p) TCP where the destination port (--dport) is 6443, and specifies the target jump (-j) rule as ACCEPT.
sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
#sudo service iptables restart
sudo systemctl restart iptables
sudo iptables-save | sudo tee -a /etc/iptables.conf
sudo iptables-restore < /etc/iptables.conf
sudo iptables -L

nc -l -p 6443
# check
telnet <IP> 6443
# ^]
# telnet> close


# sudo systemctl stop iptables
# chkconfig iptables off
# chkconfig  --list |grep iptables
# sudo systemctl disable iptables
```
### Control plane
| Protocol | Direction | Port Range | Purpose                 | Used By              |
| :------: | :-------: | :--------: | :---------------------- | :------------------- |
|   TCP    |  Inbound  |    6443    | Kubernetes API server   | All                  |
|   TCP    |  Inbound  | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
|   TCP    |  Inbound  |   10250    | Kubelet API             | Self, Control plane  |
|   TCP    |  Inbound  |   10259    | kube-scheduler          | Self                 |
|   TCP    |  Inbound  |   10257    | kube-controller-manager | Self                 |

### Worker node(s)
| Protocol | Direction | Port Range  | Purpose            | Used By             |
| :------: | :-------: | :---------: | :----------------- | :------------------ |
|   TCP    |  Inbound  |    10250    | Kubelet API        | Self, Control plane |
|   TCP    |  Inbound  | 30000-32767 | NodePort Services† | All                 |

more detailes: [Ports and Protocols](https://kubernetes.io/docs/reference/ports-and-protocols/), [Opening a port on Linux](https://www.digitalocean.com/community/tutorials/opening-a-port-on-linux)