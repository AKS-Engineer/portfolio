# NVIDIA GPU Operator — Engineering Implementation  
Automated GPU lifecycle management for Kubernetes  
Ubuntu 24.04 • containerd • CDI • NVIDIA GPUs

---

## 1. Purpose

The NVIDIA GPU Operator is the automation layer that transforms a GPU‑enabled Kubernetes node into a **self‑managing GPU platform**.  
It handles everything that would otherwise require manual engineering:

- Driver lifecycle  
- CUDA toolkit lifecycle  
- NVIDIA container runtime configuration  
- Device plugin deployment  
- DCGM + DCGM Exporter  
- MIG configuration (if supported)  
- GPU node health monitoring  

This document defines the engineering implementation of the GPU Operator in a way that reflects **platform thinking**, not just “apply a YAML.”

---

## 2. Architecture Overview

```
+---------------------------------------------------------------+
|                       GPU Operator Layer                      |
+---------------------------------------------------------------+
|  Driver  |  Toolkit  |  Device Plugin  |  DCGM  |  Exporter   |
|  MIG Manager (if supported)                                   |
+---------------------------------------------------------------+
|        Kubernetes Node (containerd + CDI + kubelet)           |
+---------------------------------------------------------------+
|        NVIDIA Driver | CUDA Runtime | DCGM (optional)         |
+---------------------------------------------------------------+
|                       Ubuntu 24.04 Host                       |
+---------------------------------------------------------------+
```

The Operator effectively becomes the **GPU control plane** inside your Kubernetes cluster.

---

## 3. Engineering Principles

1. **Operator owns the GPU lifecycle**  
   Once deployed, the Operator becomes the source of truth for GPU configuration.

2. **Requires containerd + CDI**  
   The Operator expects a modern runtime environment.  
   Docker‑shim is deprecated. CDI is the new standard.

3. **DCGM Exporter is deployed automatically**  
   This gives you GPU metrics in Prometheus without extra work.

4. **MIG is managed declaratively**  
   If your GPU supports MIG, the Operator handles partitioning.

5. **Nodes must be GPU‑ready before Operator deployment**  
   Drivers must be installed and `nvidia-smi` must work.

---

## 4. Prerequisites

Before deploying the Operator, ensure:

- `nvidia-smi` works  
- containerd is installed  
- SystemdCgroup is enabled  
- NVIDIA container toolkit is installed  
- CDI spec exists at `/etc/cdi/nvidia.yaml`  
- Node has joined the Kubernetes cluster  

This ensures the Operator doesn’t fight with pre‑existing misconfigurations.

---

## 5. Implementation Steps (with rationale)

### 5.1 Create Namespace

```
kubectl create namespace gpu-operator
```

Keeping GPU components isolated improves clarity and troubleshooting.

---

### 5.2 Deploy the GPU Operator

```
kubectl apply -f https://github.com/NVIDIA/gpu-operator/raw/v23.9.0/deployments/gpu-operator.yaml
```

This deploys:

- driver container  
- toolkit container  
- device plugin  
- DCGM  
- DCGM Exporter  
- MIG Manager  

Each component is a separate pod, giving you modular lifecycle control.

---

### 5.3 Validate Operator Pods

```
kubectl get pods -n gpu-operator
```

Expected pods:

- `nvidia-driver-daemonset`  
- `nvidia-container-toolkit-daemonset`  
- `nvidia-device-plugin-daemonset`  
- `nvidia-dcgm`  
- `nvidia-dcgm-exporter`  
- `nvidia-mig-manager` (if supported)

If any pod is CrashLooping, check:

```
kubectl logs <pod> -n gpu-operator
```

---

### 5.4 Validate GPU Scheduling

```
kubectl run gpu-op-test \
  --image=nvidia/cuda:12.2.0-base \
  --command -- nvidia-smi
```

If the Operator is healthy, this pod should show:

- GPU detected  
- Driver version  
- CUDA version  

This is the “hello world” of GPU scheduling.

---

### 5.5 Validate DCGM Exporter Metrics

```
kubectl port-forward -n gpu-operator svc/dcgm-exporter 9400:9400
curl localhost:9400/metrics
```

You should see metrics like:

- `DCGM_FI_DEV_GPU_TEMP`  
- `DCGM_FI_DEV_POWER_USAGE`  
- `DCGM_FI_DEV_SM_CLOCK`  

This is the foundation for GPU dashboards in Grafana.

---

### 5.6 MIG Configuration (If Supported)

To enable MIG:

```
nvidia-smi -mig 1
```

The Operator will:

- detect MIG mode  
- partition GPUs  
- expose MIG devices via CDI  
- update device plugin  

This is how cloud providers offer fractional GPUs.

---

## 6. Troubleshooting

### Issue: Driver pod CrashLooping  
Cause: host driver mismatch  
Fix: ensure host driver matches Operator version

### Issue: Device plugin not registering GPUs  
Cause: CDI spec missing  
Fix: regenerate `/etc/cdi/nvidia.yaml`

### Issue: DCGM Exporter missing metrics  
Cause: DCGM failed to initialize  
Fix: restart DCGM pod

---

## 7. Versioning Notes

- Operator v23.9.0 supports CUDA 12.x  
- MIG requires Ampere or newer  
- containerd 1.7+ recommended  
- Kubernetes 1.28+ recommended  

---

## 8. Integration Points

This layer integrates with:

- GPU Kubernetes Node  
- GPU Infra & Observability  
- GPU Python Compute workloads  
- Jupyter GPU pods  
- Spark GPU executors  

The Operator becomes the **automation backbone** of your GPU platform.

---
