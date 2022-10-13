# Kubernetes Cluster installation using kubeadm
[Kubeadm](https://kubernetes.io/docs/reference/setup-tools/kubeadm/) is a tool built to provide kubeadm init and kubeadm join as best-practice "fast paths" for creating Kubernetes clusters.

kubeadm performs the actions necessary to get a minimum viable cluster up and running. By design, it cares only about bootstrapping, not about provisioning machines.

<!-- TABLE OF CONTENTS -->
## Table of Contents
- [Kubernetes Cluster installation using kubeadm](#kubernetes-cluster-installation-using-kubeadm)
  - [Table of Contents](#table-of-contents)
  - [Installing kubeadm](#installing-kubeadm)
    - [System Requirements](#system-requirements)

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
```