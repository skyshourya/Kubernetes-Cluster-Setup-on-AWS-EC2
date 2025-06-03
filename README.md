
# Kubernetes Cluster Setup on AWS EC2 (Ubuntu 24.04)

## Using Kubeadm, Containerd Runtime, and Calico Networking (Kubernetes v1.29)

---

## 📌 Overview

This guide provides a comprehensive walkthrough to set up a multi-node Kubernetes cluster on AWS EC2 instances using:
- `kubeadm` for cluster initialization
- `containerd` as the container runtime
- `Calico` for networking
- Kubernetes v1.29.6 (upgrade to v1.30 can be done later)

---

## 📋 Table of Contents

- [Introduction](#introduction)
- [Architecture](#architecture)
- [AWS EC2 Security Group Configuration](#aws-ec2-security-group-configuration)
- [Master Node Setup](#master-node-setup)
- [Worker Node Setup](#worker-node-setup)
- [Join Worker to Cluster](#join-worker-to-cluster)
- [Install Calico Networking](#install-calico-networking)
- [Validation and Troubleshooting](#validation-and-troubleshooting)

---

## 🎬 Introduction

`kubeadm` is a tool to bootstrap the Kubernetes cluster, which installs all the control plane components and prepares the cluster.

---

## 📊 Architecture

- 1 Master Node
- 2 Worker Nodes (can scale more)
- EC2 Ubuntu 24.04 LTS Instances
- Internal VPC/Private IP communication

---

## 🔐 AWS EC2 Security Group Configuration

Ensure traffic on the following ports is allowed:

- TCP: 6443, 2379-2380, 10250-10252, 30000-32767
- UDP: VXLAN ports for Calico if needed (4789)

Also disable **Source/Destination check** on all instances.

---

## 🧰 Master Node Setup

### Step 1: SSH and Prepare OS

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### Step 2: Enable Kernel Modules and Sysctl

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

### Step 3: Install Container Runtime

```bash
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz

curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

---

### Step 4: Install runc and CNI Plugins

```bash
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```

---

### Step 5: Install Kubernetes Tools

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades
sudo apt-mark hold kubelet kubeadm kubectl
```

---

### Step 6: Configure crictl

```bash
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```

---

### Step 7: Initialize Kubernetes Master

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<MASTER_PRIVATE_IP>
```

Save the `kubeadm join` command for later use!

---

### Step 8: Setup Kubeconfig

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## ⚙️ Worker Node Setup

Perform Steps 1-6 from Master Node setup on each Worker.

Then join the node to the cluster using:

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

If needed:

```bash
kubeadm token create --print-join-command
```

---

## 🌐 Install Calico Networking

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml
kubectl apply -f custom-resources.yaml
```

---

## ✅ Validation and Troubleshooting

```bash
kubectl get nodes
kubectl get pods -A
```

### If Calico pods not running:

```bash
kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=interface=ens5
kubectl delete pods -n calico-system -l k8s-app=calico-node
```

Or use manifest-based install:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

---

## 🎉 Cluster Ready!

You now have a production-grade Kubernetes cluster on AWS EC2 using kubeadm, containerd, and Calico.
