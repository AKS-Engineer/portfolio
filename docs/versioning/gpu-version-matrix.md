# GPU Platform Version Matrix  
Compatibility and lifecycle contract for the GPU platform

---

## Purpose

This document defines the supported versions of all GPU‑related components in the platform.  
It acts as a **compatibility contract**, ensuring that upgrades are predictable and that all layers remain stable.

---

## Version Matrix

| Component            | Version        | Notes |
|----------------------|----------------|-------|
| **CUDA Toolkit**     | 12.5 / 13.0    | Installed via NVIDIA repo |
| **NVIDIA Driver**    | 550.xx         | Required for CUDA 12.x |
| **CuPy**             | CUDA 12.x      | Matches CUDA toolkit |
| **RAPIDS cuDF**      | 24.xx          | Requires CUDA 12.x |
| **PyTorch CUDA**     | 2.2 (cu121)    | Matches CUDA 12.1 |
| **GPU Operator**     | v23.9.0        | Automates driver/toolkit/DCGM |
| **Device Plugin**    | v0.16.0        | CDI‑compatible |
| **Kubernetes**       | 1.28+          | Required for GPU Operator |
| **containerd**       | 1.7+           | Required for CDI support |

---

## Upgrade Strategy

### **1. Bottom‑up upgrades**
Always upgrade in this order:

1. GPU Operator  
2. Drivers  
3. CUDA toolkit  
4. Python GPU libraries  
5. Interactive compute stack  

### **2. Version pinning**
All components should be pinned to explicit versions to avoid drift.

### **3. Compatibility testing**
Each upgrade should validate:

- CUDA compatibility  
- PyTorch / RAPIDS compatibility  
- Device plugin registration  
- DCGM metrics  
- Jupyter GPU visibility  

---

## Summary

This version matrix ensures that the GPU platform remains stable, reproducible, and predictable across upgrades.  
It defines the lifecycle contract for all GPU‑related components and prevents runtime drift.

---
