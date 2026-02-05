# GPU Kubernetes Node — Engineering Implementation

This document covers containerd, CDI, kubeadm, and NVIDIA device plugin setup for GPU scheduling.

## 1. Pre‑Checks

nvidia-smi

## 2. Install containerd

sudo apt update
sudo apt install -y containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

## 3. Install NVIDIA Container Toolkit (CDI)

sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=containerd
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo systemctl restart containerd

## 4. Install Kubernetes Components

sudo apt install -y kubeadm kubelet kubectl
sudo systemctl enable kubelet
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

## 5. Cluster Init or Join

sudo kubeadm init --pod-network-cidr=192.168.0.0/16  
# or join using control-plane token

## 6. Deploy NVIDIA Device Plugin

kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.16.0/nvidia-device-plugin.yml

## 7. Validation

kubectl describe node <node> | grep -A5 Capacity
kubectl run gpu-test --image=nvidia/cuda:12.2.0-base --command -- nvidia-smi
