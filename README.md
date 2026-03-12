


# Kubernetes Cluster Setup Guide

![Build Status](https://img.shields.io/badge/build-passing-brightgreen) ![License](https://img.shields.io/badge/license-MIT-blue) ![Kubernetes Version](https://img.shields.io/badge/kubernetes-v1.35-blue)

## Project Overview
This repository contains the setup documentation and automation scripts for deploying an on-premises Kubernetes cluster. The cluster is designed to provide a highly available and scalable foundation for hosting micro-SaaS applications, ensuring a reliable infrastructure for a tight-knit development team of three to build, test, and deploy services efficiently.

## Architecture Overview


The deployment consists of a single master node for the control plane and multiple worker nodes for scheduling workloads. Pod-to-pod communication is handled by the Calico network plugin. External traffic routing is managed via a combination of MetalLB for bare-metal LoadBalancer services and the NGINX Ingress Controller. Distributed persistent block storage is provided by Longhorn.

## Prerequisites
* **Hardware / VMs**: One master node and three worker nodes running Linux.
* **Network**: Nodes must be reachable over the network and have the following statically assigned IPs and hostnames:
    * Master 01: `192.168.172.51` (Hostname: `k8s-master-01`).
    * Worker 01: `192.168.172.52` (Hostname: `k8s-worker-01`).
    * Worker 02: `192.168.172.53` (Hostname: `k8s-worker-02`).
    * Worker 03: `192.168.172.54` (Hostname: `k8s-worker-03`).
* **Privileges**: `sudo` access on all machines to execute the setup scripts.
---

## Step-by-Step Deployment Guide

### 1. Initial Node Setup (All Nodes)
Create a new setup script to configure both the master and worker nodes. Create the file, make it executable, and open it for editing:
```bash
touch kubernetes-cluster-setup.sh
chmod +x kubernetes-cluster-setup.sh
nano kubernetes-cluster-setup.sh

```

Paste the following shell script into the file and execute it to disable swap, configure networking, and install `kubeadm`, `kubelet`, `kubectl`, and the `containerd` runtime:

```bash
#!/bin/bash 
set -e
echo "[Step 1] Updating /etc/hosts ..."
cat <<EOF | sudo tee -a /etc/hosts
192.168.172.51        k8s-master-01
192.168.172.52        k8s-worker-01
192.168.172.53        k8s-worker-02
192.168.172.54        k8s-worker-03
EOF

echo "[Step 2] Disabling swap ..."
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "[Step 3] Configuring sysctl ..."
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
sysctl net.ipv4.ip_forward

echo "[Step 4] Installing Kubernetes packages ..."
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.35/deb/](https://pkgs.k8s.io/core:/stable:/v1.35/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "[Step 5] Removing old container runtimes ..."
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do      
    sudo apt-get remove -y $pkg || true 
done 

echo "[Step 6] Installing containerd runtime ..."
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
[https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install -y containerd.io
sudo mkdir -p /etc/containerd
sudo sh -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

echo "[Step 7] Restarting services ..."
sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
echo "[✔] Worker node setup completed. Now run 'kubeadm join ...' from the master node output."
```

### 2. Initialize the Control Plane (Master Node Only)

Initialize `kubeadm` exclusively on the master nodes. Pull the required images and initialize the cluster, explicitly defining the pod network CIDR:

```bash
sudo kubeadm config images pull
sudo kubeadm init --pod-network-cidr=10.10.0.0/16 
```

Next, copy the generated configuration file to the local user’s directory on the master nodes to enable `kubectl` commands:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config 
```

### 3. Configure the Calico Network (Master Node Only)

Setup the Calico network strictly on the master nodes to facilitate pod-to-pod communication. Execute the following to create the operator and apply custom resources with the updated CIDR matching the `kubeadm init` step:

```bash
kubectl create -f [https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/tigera-operator.yaml)
curl [https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml](https://raw.githubusercontent.com/projectcalico/calico/v3.28.1/manifests/custom-resources.yaml) -O
sed -i 's/cidr: 192\.168\.0\.0\/16/cidr: 10.10.0.0\/16/g' custom-resources.yaml
kubectl create -f custom-resources.yaml 

```

### 4. Join Worker Nodes (Worker Nodes Only)

After the master node is successfully initialized, you can join the worker nodes to the cluster. Utilize the token provided at the end of the `kubeadm init` output. The token output will resemble the following format:

```bash
kubeadm join 192.168.172.51:6443 --token 9dimka.v2waaejgl5q53xvl \
        --discovery-token-ca-cert-hash sha256:6bec834acc0c257832c3a7c175f26ea7912488ed38ba029a9d4b82cd5b65ce2e
```

Copy your specific token and execute it on all worker nodes to successfully join them to the newly created cluster.

### 5. Setup MetalLB (Master Node Only)

Deploy MetalLB on the master nodes to provide network load balancing. First, apply the native manifests:

```bash
kubectl apply -f [https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml](https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml) 
```

Create a MetalLB configuration file named `metallb-config.yaml` on the master nodes:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-address-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.172.240-192.168.172.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - default-address-pool 
```

Apply the created configuration file and confirm the resources are created:

```bash
kubectl apply -f metallb-config.yaml
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system 
```

### 6. Install Ingress Controller (Master Node Only)

Install the Ingress controller on the master nodes to route external HTTP/HTTPS traffic. Deploy the `ingress-nginx` stack, which defaults to a Service of type LoadBalancer:

```bash
kubectl apply -f [https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml](https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml) 
```

Verify that the Ingress controller is running and has successfully acquired an external IP address:

```bash
kubectl -n ingress-nginx get pods -o wide
kubectl -n ingress-nginx get svc ingress-nginx-controller
kubectl -n ingress-nginx describe svc ingress-nginx-controller 
```

### 7. Configure Longhorn Storage (All Nodes)

Prepare all nodes (Master and Worker) prior to deploying Longhorn. On every node designated to store Longhorn data, install the required `open-iscsi` and `nfs-common` packages:

```bash
sudo apt-get update
sudo apt-get install -y open-iscsi nfs-common
sudo systemctl enable --now iscsid || sudo systemctl enable --now open-iscsi 
```

Next, apply the Longhorn manifests exclusively on the master nodes. This command will deploy Longhorn into its own dedicated namespace:

```bash
kubectl create namespace longhorn-system
kubectl apply -f [https://raw.githubusercontent.com/longhorn/longhorn/v1.11.1/deploy/longhorn.yaml](https://raw.githubusercontent.com/longhorn/longhorn/v1.11.1/deploy/longhorn.yaml) 
```

Watch the pods initialize and come online on the master nodes:

```bash
kubectl -n longhorn-system get pods 
```
## Validation

After ensuring all nodes have successfully joined the cluster, verify that the environment is fully operational by executing the following commands on the master node:

```bash
kubectl get no
kubectl get po -A 
```
