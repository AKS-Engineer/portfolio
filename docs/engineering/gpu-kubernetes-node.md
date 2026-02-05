# GPU Kubernetes Node — Engineering Implementation
containerd • CDI • kubeadm • NVIDIA Device Plugin  
Ubuntu 24.04

---

## 1. Purpose

This document defines the engineering steps to prepare a **GPU‑enabled Kubernetes node** using containerd, NVIDIA container toolkit, CDI, and the NVIDIA device plugin.

This is the layer that turns a GPU host into a **GPU‑schedulable Kubernetes worker**.

---

## 2. Architecture Overview

```
+-------------------------------------------------------------+
|                   Kubernetes GPU Node Layer                 |
+-------------------------------------------------------------+
| NVIDIA Device Plugin | CDI | containerd GPU Runtime         |
+-------------------------------------------------------------+
| NVIDIA Driver | CUDA Runtime | DCGM (optional)              |
+-------------------------------------------------------------+
| Ubuntu 24.04 Host                                           |
+-------------------------------------------------------------+
```

---

## 3. Engineering Principles

1. **containerd must use SystemdCgroup** for kubelet compatibility.  
2. **NVIDIA container toolkit must run in CDI mode** (modern standard).  
3. **CDI spec must be generated** for GPU enumeration.  
4. **Device plugin is deployed after node joins cluster.**

---

## 4. Implementation Steps (with rationale)

### 4.1 Pre‑Checks

```
nvidia-smi
```

Driver must be installed before Kubernetes GPU setup.

### 4.2 Install containerd

```
sudo apt update
sudo apt install -y containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

### 4.3 Install NVIDIA Container Toolkit (CDI Mode)

```
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=containerd
sudo mkdir -p /etc/cdi
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo systemctl restart containerd
```

### 4.4 Install Kubernetes Components

```
sudo apt install -y kubeadm kubelet kubectl
sudo systemctl enable kubelet
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```

### 4.5 Cluster Init or Join

Control plane:

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

Worker:

Use join token from control plane.

### 4.6 Deploy NVIDIA Device Plugin

```
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.16.0/nvidia-device-plugin.yml
```

### 4.7 Validation

```
kubectl describe node <node> | grep -A5 Capacity
kubectl run gpu-test --image=nvidia/cuda:12.2.0-base --command -- nvidia-smi
```

---

## 5. Troubleshooting

### Issue: GPU pod fails with “no CDI devices found”
Cause: CDI spec missing  
Fix: regenerate `/etc/cdi/nvidia.yaml`

### Issue: kubelet fails to start
Cause: SystemdCgroup not enabled  
Fix: update containerd config

---

## 6. Versioning Notes

- Device plugin v0.16.0 supports CUDA 12.x  
- containerd 1.7+ recommended  
- kubeadm 1.30+ recommended  

---

## 7. Integration Points

This layer feeds into:

- GPU Operator  
- GPU‑aware workloads  
- Jupyter GPU pods  
- Spark GPU executors  

---
