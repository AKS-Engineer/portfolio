# GPU Infrastructure & Observability — Engineering Implementation

This document covers DCGM, CUDA toolkit installation, NVIDIA repo pinning, diagnostics, and GPU observability on Ubuntu 24.04.

## 1. Add NVIDIA CUDA Repository

wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-ubuntu2404.pin
sudo mv cuda-ubuntu2404.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/3bf863cc.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/ /"
sudo apt update

## 2. Install DCGM

sudo apt install -y datacenter-gpu-manager
sudo systemctl enable --now nvidia-dcgm

## 3. GPU Discovery

dcgmi discovery -l

## 4. Health Checks

dcgmi health -c
dcgmi health -g 0 -c

## 5. Install CUDA Toolkit (12.5 or 13.0)

sudo apt install -y cuda-toolkit-13-0
# or
sudo apt install -y cuda-toolkit-12-5

## 6. Add CUDA to PATH

export PATH=/usr/local/cuda/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH

## 7. Verify

nvcc --version
dcgmi diag -r 3

## 8. Build CUDA Samples

sudo apt install -y cmake
git clone https://github.com/NVIDIA/cuda-samples.git
cd cuda-samples
mkdir build && cd build
cmake ..
make -j$(nproc)
./Samples/1_Utilities/deviceQuery/deviceQuery
