# GPU Interactive Compute — Engineering Implementation  
Jupyter • RAPIDS • PyTorch • Spark RAPIDS Accelerator  
Ubuntu 24.04 • NVIDIA GPU • CUDA 12.x/13.x

---

## 1. Purpose

This document defines the **interactive GPU compute layer** of the platform.  
Where the previous documents focused on:

- GPU Python compute (local environment)
- GPU infra + observability
- GPU Kubernetes node
- GPU Operator

…this layer focuses on **how developers actually interact with GPUs** through:

- Jupyter GPU kernels  
- RAPIDS notebooks  
- PyTorch notebooks  
- Optional Spark RAPIDS GPU acceleration  

This is the “developer experience” layer of the GPU platform.

---

## 2. Architecture Overview

```
+---------------------------------------------------------------+
|                   Interactive GPU Compute Layer               |
+---------------------------------------------------------------+
| Jupyter GPU Kernel | RAPIDS | PyTorch | Spark RAPIDS (opt.)  |
+---------------------------------------------------------------+
| GPU Python Compute Stack (CuPy, cuDF, PyTorch CUDA)           |
+---------------------------------------------------------------+
| CUDA Runtime | NVRTC | CUDA Toolkit | NVIDIA Driver           |
+---------------------------------------------------------------+
| Ubuntu 24.04 Host or Kubernetes GPU Pod                       |
+---------------------------------------------------------------+
```

This layer sits **on top of** the GPU Python compute stack and **exposes GPU acceleration to users** in a notebook‑friendly way.

---

## 3. Engineering Principles

1. **Interactive compute must be isolated**  
   Each GPU kernel runs in its own environment to avoid dependency collisions.

2. **GPU acceleration must be transparent**  
   Developers should not need to understand CUDA internals.

3. **Jupyter must detect GPUs reliably**  
   Kernel registration must point to the correct Python environment.

4. **Spark GPU acceleration is optional**  
   It is included only when the platform needs distributed GPU compute.

---

## 4. Implementation Steps (with rationale)

### 4.1 Install Jupyter GPU Kernel

This kernel is created from the GPU Python compute environment.

```
python -m ipykernel install \
  --prefix=/opt/jupyter/data \
  --name gpu-env \
  --display-name "Python (gpu-env)"
```

This ensures:

- Jupyter sees the GPU environment  
- Kernel is isolated from system Python  
- Notebook users get GPU acceleration automatically  

---

### 4.2 Validate GPU Kernel

Inside a notebook:

```python
import cupy as cp; cp.arange(10)
import torch; torch.cuda.is_available()
import cudf; cudf.Series([1,2,3])
```

Expected:

- CuPy returns a GPU array  
- PyTorch returns `True`  
- cuDF constructs a GPU Series  

This confirms the entire GPU Python stack is wired correctly.

---

### 4.3 Optional: Spark RAPIDS Accelerator

If the platform supports distributed compute, install the RAPIDS Accelerator:

```
wget https://repo1.maven.org/maven2/com/nvidia/rapids-4-spark_2.12/24.02.0/rapids-4-spark_2.12-24.02.0.jar
```

Spark configuration:

```
spark.plugins=com.nvidia.spark.SQLPlugin
spark.rapids.sql.enabled=true
spark.executor.resource.gpu.amount=1
spark.task.resource.gpu.amount=0.125
```

This enables:

- GPU‑accelerated DataFrame operations  
- GPU‑accelerated SQL  
- GPU‑accelerated ETL pipelines  

---

### 4.4 Validate Spark GPU Acceleration

```python
from pyspark.sql import SparkSession
spark = SparkSession.builder.getOrCreate()

df = spark.range(1_000_000)
df.selectExpr("sum(id)").show()
```

GPU acceleration is visible in Spark logs:

- SQLPlugin enabled  
- GPU tasks scheduled  
- RapidsShuffleManager active  

---

## 5. Troubleshooting

### Issue: Jupyter kernel not showing GPU  
Cause: kernel registered to wrong Python environment  
Fix: reinstall kernel with correct prefix

### Issue: cuDF import fails  
Cause: missing NVRTC or CUDA headers  
Fix: reinstall `cuda-toolkit-12-5` or `cuda-nvrtc-12-5`

### Issue: Spark not using GPU  
Cause: missing RAPIDS jar or plugin config  
Fix: verify Spark config and jar path

---

## 6. Versioning Notes

- Jupyter kernels are environment‑specific  
- RAPIDS versions track CUDA minor versions  
- Spark RAPIDS requires Spark 3.3+  
- PyTorch CUDA wheels track CUDA 12.x  

---

## 7. Integration Points

This layer integrates with:

- GPU Python Compute Stack  
- GPU Kubernetes Node (for JupyterHub on K8s)  
- GPU Operator (for GPU‑aware pods)  
- GPU Infra & Observability (DCGM metrics for notebooks)  

This is the layer developers touch directly — the “face” of your GPU platform.

---
