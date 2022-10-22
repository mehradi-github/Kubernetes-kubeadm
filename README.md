# Kubernetes Cluster installation using kubeadm
[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool built to provide kubeadm init and kubeadm join as best-practice "fast paths" for creating Kubernetes clusters.

kubeadm performs the actions necessary to get a minimum viable cluster up and running. By design, it cares only about bootstrapping, not about provisioning machines.

<!-- TABLE OF CONTENTS -->
## Table of Contents
- [Kubernetes Cluster installation using kubeadm](#kubernetes-cluster-installation-using-kubeadm)
  - [Table of Contents](#table-of-contents)
  - [Installing Kubernetes Cluster using kubeadm](#installing-kubernetes-cluster-using-kubeadm)
    - [System Requirements](#system-requirements)
    - [Forwarding IPv4 and letting iptables see bridged traffic](#forwarding-ipv4-and-letting-iptables-see-bridged-traffic)
    - [Installing Docker as container runtime (CRI)](#installing-docker-as-container-runtime-cri)
    - [Installing kubeadm, kubelet and kubectl](#installing-kubeadm-kubelet-and-kubectl)
    - [Configuring cgroups driver (Ignore if runtime is docker)](#configuring-cgroups-driver-ignore-if-runtime-is-docker)
    - [Initializing Kubernetes Cluster(on master node)](#initializing-kubernetes-clusteron-master-node)
    - [Installing Calico networking and network policy(on master node)](#installing-calico-networking-and-network-policyon-master-node)
    - [Joining the worker nodes to the master node to from the k8s cluster(on worker node)](#joining-the-worker-nodes-to-the-master-node-to-from-the-k8s-clusteron-worker-node)
    - [Accessing the K8s Cluster and Deploying a POD(on master node)](#accessing-the-k8s-cluster-and-deploying-a-podon-master-node)
  - [kubectl Cheat Sheet](#kubectl-cheat-sheet)
  - [Ingress NGINX Controller](#ingress-nginx-controller)

## Installing Kubernetes Cluster using kubeadm
This page shows how to [install the kubeadm toolbox](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) on Amazon Linux 2 as a virtual machine.

### System Requirements
- A compatible Linux host. The Kubernetes project provides generic instructions for Linux distributions based on Debian and Red Hat, and those distributions without a package manager.
- 2 GB or more of RAM per machine (any less will leave little room for your apps).
- 2 CPUs or more.
- Full network connectivity between all machines in the cluster (public or private network is fine).
- Unique hostname, MAC address, and product_uuid for every node.
- Swap disabled. You MUST disable swap in order for the kubelet to work properly.

```sh
dhclient -v
ip link 
ifconfig -a
sudo cat /sys/class/dmi/id/product_uuid

hostnamectl set-hostname 'gfs-server-03'
hostnamectl

sudo swapoff -a
# Disabling firewall
sudo systemctl disable --now iptables

# # Check required ports
# sudo yum install nc
# sudo yum install iptables-services

# sudo systemctl enable iptables
# sudo systemctl start iptables
# sudo systemctl status iptables

# #List all open ports
# # This will print all listening sockets (-l) along with the port number (-n), with TCP ports (-t) and UDP ports (-u) also listed in the output.
# netstat -lntu

# #list listening sockets with an open port
# ss -lntu

# # This sets the firewall to append (-A) the new rule to accept input packets via protocol (-p) TCP where the destination port (--dport) is 6443, and specifies the target jump (-j) rule as ACCEPT.
# sudo iptables -A INPUT -p tcp --dport 6443 -j ACCEPT
# #sudo service iptables restart
# sudo systemctl restart iptables
# sudo iptables-save | sudo tee -a /etc/iptables.conf
# sudo iptables-restore < /etc/iptables.conf
# sudo iptables -L

# nc -l -p 6443
# # check
# telnet <IP> 6443
# # ^]
# # telnet> close


# sudo systemctl stop iptables
# chkconfig iptables off
# chkconfig  --list |grep iptables
# sudo systemctl disable iptables
```
**Control plane**
| Protocol | Direction | Port Range | Purpose                 | Used By              |
| :------: | :-------: | :--------: | :---------------------- | :------------------- |
|   TCP    |  Inbound  |    6443    | Kubernetes API server   | All                  |
|   TCP    |  Inbound  | 2379-2380  | etcd server client API  | kube-apiserver, etcd |
|   TCP    |  Inbound  |   10250    | Kubelet API             | Self, Control plane  |
|   TCP    |  Inbound  |   10259    | kube-scheduler          | Self                 |
|   TCP    |  Inbound  |   10257    | kube-controller-manager | Self                 |

**Worker node(s)**
| Protocol | Direction | Port Range  | Purpose            | Used By             |
| :------: | :-------: | :---------: | :----------------- | :------------------ |
|   TCP    |  Inbound  |    10250    | Kubelet API        | Self, Control plane |
|   TCP    |  Inbound  | 30000-32767 | NodePort Services† | All                 |

more detailes: [Ports and Protocols](https://kubernetes.io/docs/reference/ports-and-protocols/), [Opening a port on Linux](https://www.digitalocean.com/community/tutorials/opening-a-port-on-linux)

### Forwarding IPv4 and letting iptables see bridged traffic

```sh

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
#To load it explicitly
sudo modprobe br_netfilter

#Verify that the br_netfilter module is loaded 
lsmod | grep br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system


```

### Installing Docker as container runtime (CRI)
To run containers in Pods, Kubernetes uses a [container runtime](https://kubernetes.io/docs/setup/production-environment/container-runtimes/).
By default, Kubernetes uses the Container Runtime Interface (CRI) to interface with your chosen container runtime.

[Docker Engine](https://docs.docker.com/engine/install/) is available on a variety of Linux platforms.
The contents of /var/lib/docker/, including images, containers, volumes, and networks, are preserved. The Docker Engine package is now called **docker-ce**.
```sh

#Uninstall old versions
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine

# installing by amazon-linux-extras   
sudo yum -y update
sudo yum install -y amazon-linux-extras
amazon-linux-extras | grep -n -i docker
sudo amazon-linux-extras enable docker
sudo yum install docker -y


# Installing by dnf 
# sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
# sudo dnf update
# sudo dnf install -y docker-ce docker-ce-cli containerd.io
# sudo dnf install docker-ce docker-ce-cli containerd.io --allowerasing -y
# Disabling repo
# yum repolist all
# sudo yum-config-manager --disable docker-ce-stable

# Installing by yum 
# sudo yum install -y yum-utils
# sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo    
# sudo yum install docker-ce docker-ce-cli containerd.io

# start docker services
sudo systemctl enable docker
sudo systemctl start docker
service docker status
sudo docker version

sudo useradd dockeradmin
sudo passwd dockeradmin
sudo usermod -aG docker dockeradmin
 
```
### Installing kubeadm, kubelet and kubectl

- kubeadm: the command to bootstrap the cluster.

- kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.

- kubectl: the command line util to talk to your cluster.

```sh
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
### Configuring cgroups driver (Ignore if runtime is docker)
On Linux, control groups are used to constrain resources that are allocated to processes.
Both kubelet and the underlying container runtime need to interface with control groups to enforce resource management for pods and containers and set resources such as cpu/memory requests and limits. To interface with control groups, the kubelet and the container runtime need to use a cgroup driver. It's critical that the kubelet and the container runtime uses the same cgroup driver and are configured the same.
There are two [cgroup drivers](https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cgroup-drivers) available:
- cgroupfs
- systemd
  
### Initializing Kubernetes Cluster(on master node)
```sh
sudo su
ip addr
kubeadm init --apiserver-advertise-address=<MasterServerIP> --pod-network-cidr=192.168.0.0/16
exit
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```
### Installing Calico networking and network policy(on master node)
[Calico](https://projectcalico.docs.tigera.io/about/about-calico) is a [networking and network policy](https://kubernetes.io/docs/concepts/cluster-administration/addons/#networking-and-network-policy) provider. Calico supports a flexible set of networking options so you can choose the most efficient option for your situation, including non-overlay and overlay networks, with or without BGP. Calico uses the same engine to enforce network policy for hosts, pods, and (if using Istio & Envoy) applications at the service mesh layer.
[Install Calico with Kubernetes API datastore, 50 nodes or less](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less)
```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml

# kubeadm token create --print-join-command
```
### Joining the worker nodes to the master node to from the k8s cluster(on worker node)
```sh
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
        --discovery-token-ca-cert-hash <HASH>

```

### Accessing the K8s Cluster and Deploying a POD(on master node)
```sh
# Get component status
kubectl get cs
# Get nodes
kubectl get nodes -o wide

#kubectl run mynginx --image=nginx 
kubectl delete deployment nginx-dep
kubectl apply -f nginx-deployment.yaml
kubectl get pods -o wide --watch
kubectl get services -o wide 
kubectl describe service nginx-srv
curl <NODE_IP>:<NodeProt>

```
## kubectl Cheat Sheet
This page contains a list of commonly used kubectl commands and flags. [**kubectl Cheat Sheet**](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

## Ingress NGINX Controller
ingress-nginx is an Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer.