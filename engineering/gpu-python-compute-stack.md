# GPU Python Compute Stack — Engineering Implementation

This document covers the GPU‑accelerated Python environment used for RAPIDS, CuPy, PyTorch, and Jupyter GPU kernels on Ubuntu 24.04.

## 1. Install GPU Python Libraries

pip install cupy-cuda12x
pip install cudf-cu12 --extra-index-url=https://pypi.nvidia.com
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121

## 2. Install Jupyter GPU Kernel

pip install ipykernel
python -m ipykernel install \
  --prefix=/opt/jupyter/data \
  --name gpu-env \
  --display-name "Python (gpu-env)"

## 3. Install CUDA Runtime + NVRTC + Headers

sudo apt update
sudo apt install -y cuda-libraries-12-5
sudo apt install -y cuda-nvrtc-12-5
sudo apt install -y cuda-toolkit-12-5

## 4. Register CUDA Libraries

CUDA_LIB_PATH="/usr/local/cuda-12.5/targets/x86_64-linux/lib"
echo "$CUDA_LIB_PATH" | sudo tee /etc/ld.so.conf.d/cuda-12.conf
sudo ldconfig

## 5. Remove Conflicting Wheel NVRTC

rm -rf pygpu-env/lib/python3.12/site-packages/nvidia/cuda_nvrtc || true

## 6. Restart Jupyter

sudo systemctl restart jupyter

## 7. Validation

import cupy as cp; cp.arange(10)
import torch; torch.cuda.is_available()
import cudf; cudf.Series([1,2,3])

## 8. Key Insight

nvidia-smi shows the driver CUDA runtime.  
Python GPU libraries require CUDA runtime + headers + NVRTC.  
Ubuntu’s nvidia-cuda-toolkit is sufficient for Python GPU workloads.
