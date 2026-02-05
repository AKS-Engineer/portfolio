# GPU Infrastructure & Observability — Engineering Implementation
DCGM • CUDA Toolkit • Diagnostics • NVIDIA Repo  
Ubuntu 24.04

---

## 1. Purpose

This document defines the **infrastructure‑grade GPU layer**:  
DCGM, CUDA toolkit, diagnostics, and GPU observability.

This layer is for **platform engineers**, not end‑users.  
It enables:

- GPU health monitoring  
- GPU telemetry  
- CUDA toolkit compilation  
- MIG support (if hardware supports it)  
- GPU Operator readiness  

---

## 2. Architecture Overview

```
+-------------------------------------------------------------+
|                GPU Infra & Observability Layer              |
+-------------------------------------------------------------+
| DCGM | DCGM Exporter | CUDA Toolkit | nvcc | Diagnostics    |
+-------------------------------------------------------------+
| NVIDIA Driver (kernel module + runtime)                     |
+-------------------------------------------------------------+
| Ubuntu 24.04 Host                                           |
+-------------------------------------------------------------+
```

---

## 3. Engineering Principles

1. **Use NVIDIA’s official CUDA repo** for toolkit 12.5/13.0.  
2. **DCGM must match the installed toolkit version.**  
3. **Consumer GPUs skip hardware tests** — expected behavior.  
4. **Toolkit upgrades require DCGM reinstall.**

---

## 4. Implementation Steps (with rationale)

### 4.1 Add NVIDIA CUDA Repository

```
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /"
sudo apt update
```

### 4.2 Install DCGM

```
sudo apt install -y datacenter-gpu-manager
sudo systemctl enable --now nvidia-dcgm
```

### 4.3 GPU Discovery

```
dcgmi discovery -l
```

### 4.4 Health Checks

```
dcgmi health -c
dcgmi health -g 0 -c
```

Consumer GPUs will skip hardware tests — this is normal.

### 4.5 Install CUDA Toolkit (12.5 or 13.0)

```
sudo apt install -y cuda-toolkit-13-0
# or
sudo apt install -y cuda-toolkit-12-5
```

### 4.6 Add CUDA to PATH

```
export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
```

### 4.7 Verify

```
nvcc --version
dcgmi diag -r 3
```

### 4.8 Build CUDA Samples

```
sudo apt install -y cmake
git clone https://github.com/NVIDIA/cuda-samples.git
cd cuda-samples
mkdir build && cd build
cmake ..
make -j$(nproc)
./Samples/1_Utilities/deviceQuery/deviceQuery
```

---

## 5. Troubleshooting

### Issue: DCGM fails to start  
Cause: toolkit mismatch  
Fix: reinstall DCGM after toolkit upgrade

### Issue: `nvcc` missing  
Cause: PATH not updated  
Fix: export PATH=/usr/local/cuda/bin

---

## 6. Versioning Notes

- Ubuntu ships CUDA 12.0  
- NVIDIA repo provides 12.5 and 13.0  
- DCGM must match toolkit version  
- CUDA samples removed `bandwidthTest` in 12.5+

---

## 7. Integration Points

This layer feeds into:

- GPU Operator  
- DCGM Exporter  
- Kubernetes GPU node health  
- MIG partitioning  

---
