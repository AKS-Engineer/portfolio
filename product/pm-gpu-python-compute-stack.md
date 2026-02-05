# Product Spec: GPU Python Compute Stack  
Platform Abstraction Document

---

## 1. Purpose

The GPU Python Compute Stack provides a **consistent, reproducible, GPU‑accelerated Python environment** for data science and ML workloads.  
It abstracts away CUDA runtime complexity and ensures developers can use CuPy, RAPIDS cuDF, and PyTorch without managing drivers, toolkits, or NVRTC.

This is the foundational “developer‑facing compute layer” of the platform.

---

## 2. Problem / Context

GPU Python environments are notoriously fragile:

- CUDA versions must match library wheels  
- NVRTC mismatches cause runtime failures  
- Jupyter kernels often point to the wrong environment  
- Developers frequently break environments by mixing pip/conda/system libs  

Without a platform abstraction, every user ends up debugging CUDA instead of building models.

---

## 3. Platform Abstraction

**The platform guarantees:**

- A GPU‑accelerated Python environment that “just works”  
- Pre‑validated versions of CuPy, cuDF, PyTorch  
- NVRTC and CUDA headers installed system‑wide  
- A Jupyter kernel bound to the GPU environment  
- A stable upgrade path for CUDA minor versions  

**Developers do not manage:**

- CUDA toolkit installation  
- NVRTC compatibility  
- Library version alignment  
- ldconfig paths  
- GPU diagnostics  

The platform owns all of that.

---

## 4. Conceptual Architecture

```
Developer Notebook
        │
        ▼
GPU Python Compute Stack
(CuPy, cuDF, PyTorch, Jupyter GPU Kernel)
        │
        ▼
CUDA Runtime + NVRTC + Headers
        │
        ▼
NVIDIA Driver
```

---

## 5. User Experience

Developers experience:

- A single Jupyter kernel named **Python (gpu-env)**  
- GPU acceleration available by default  
- No CUDA configuration required  
- No environment breakage from mismatched versions  

The experience is intentionally boring — it always works.

---

## 6. Responsibilities & Boundaries

### Platform Responsibilities
- Maintain GPU Python environment  
- Validate library compatibility  
- Provide Jupyter GPU kernel  
- Ensure CUDA runtime availability  

### Developer Responsibilities
- Use the provided kernel  
- Avoid installing conflicting CUDA components  
- Report environment issues  

---

## 7. Non‑Goals

- Not a full ML platform  
- Not a distributed compute system  
- Not a replacement for conda environments  
- Not a GPU scheduler  

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| CUDA minor version drift | Pin toolkit + validate wheels |
| User installs conflicting libs | Immutable base environment |
| Jupyter kernel misbinding | Kernel installed with explicit prefix |

---

## 9. Future Extensions

- GPU‑accelerated JupyterHub  
- Pre‑built RAPIDS notebooks  
- GPU‑accelerated ETL pipelines  
- Integration with Spark GPU executors  

---
