# GPU Python Compute Stack — Engineering Implementation
A foundational layer for GPU‑accelerated data science and ML workloads  
Ubuntu 24.04 • NVIDIA GPU • CUDA 12.x/13.x Compatible

---

## 1. Purpose

This document defines the engineering implementation of the **GPU Python Compute Stack**: CuPy, RAPIDS cuDF, PyTorch CUDA, and Jupyter GPU kernels.  
This layer is intentionally **independent of Kubernetes** and represents the “developer‑facing GPU experience” in your platform.

The goal is simple:  
> A Python environment where GPU acceleration “just works” without requiring the full CUDA developer toolkit.

---

## 2. Architecture Overview

```
+-------------------------------------------------------------+
|                    Python GPU Compute Layer                 |
+-------------------------------------------------------------+
| CuPy | cuDF (RAPIDS) | PyTorch CUDA | Jupyter GPU Kernel    |
+-------------------------------------------------------------+
| CUDA Runtime | CUDA Headers | NVRTC | cuBLAS/cuFFT/cuSPARSE |
+-------------------------------------------------------------+
| NVIDIA Driver (nvidia-smi)                                  |
+-------------------------------------------------------------+
| Ubuntu 24.04 Host                                           |
+-------------------------------------------------------------+
```

Key design principle:  
**Python GPU libraries depend on the CUDA runtime + headers + NVRTC, not the full CUDA toolkit.**

---

## 3. Engineering Principles

1. **Use Ubuntu’s CUDA toolkit packages**, not the 3.8GB NVIDIA installer.  
2. **Match Python GPU libraries to CUDA runtime**, not vice‑versa.  
3. **Ensure NVRTC is installed system‑wide**, not from wheel bundles.  
4. **Register CUDA libs with ldconfig** to avoid runtime loader failures.  
5. **Keep Jupyter kernels isolated** from system Python.

---

## 4. Implementation Steps (with rationale)

### 4.1 Install GPU Python Libraries

These wheels are pre‑built against CUDA 12.x.

```
pip install cupy-cuda12x
pip install cudf-cu12 --extra-index-url=https://pypi.nvidia.com
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

### 4.2 Install Jupyter GPU Kernel

```
pip install ipykernel
python -m ipykernel install \
  --prefix=/opt/jupyter/data \
  --name gpu-env \
  --display-name "Python (gpu-env)"
```

### 4.3 Install CUDA Runtime + NVRTC + Headers

These packages provide everything Python GPU libraries need:

```
sudo apt update
sudo apt install -y cuda-libraries-12-5
sudo apt install -y cuda-nvrtc-12-5
sudo apt install -y cuda-toolkit-12-5
```

### 4.4 Register CUDA Libraries

```
CUDA_LIB_PATH="/usr/local/cuda-12.5/targets/x86_64-linux/lib"
echo "$CUDA_LIB_PATH" | sudo tee /etc/ld.so.conf.d/cuda-12.conf
sudo ldconfig
```

### 4.5 Remove Conflicting Wheel‑Bundled NVRTC

```
rm -rf pygpu-env/lib/python3.12/site-packages/nvidia/cuda_nvrtc || true
```

### 4.6 Restart Jupyter

```
sudo systemctl restart jupyter
```

---

## 5. Validation

Inside Jupyter:

```python
import cupy as cp; cp.arange(10)
import torch; torch.cuda.is_available()
import cudf; cudf.Series([1,2,3])
```

---

## 6. Troubleshooting

### Issue: `libnvrtc.so not found`
Cause: wheel‑bundled NVRTC conflicts  
Fix: remove wheel NVRTC + reinstall system NVRTC

### Issue: PyTorch sees GPU but CuPy doesn’t
Cause: mismatched CUDA minor version  
Fix: reinstall `cupy-cuda12x`

---

## 7. Versioning Notes

- Ubuntu 24.04 ships CUDA 12.0  
- Python GPU libs expect CUDA 12.1–12.5  
- Minor version mismatches are tolerated  
- NVRTC must match the toolkit version

---

## 8. Integration Points

This layer feeds into:

- GPU JupyterHub  
- GPU Spark  
- GPU Operator (user workloads)  
- Kubernetes GPU pods (Python workloads)

---
