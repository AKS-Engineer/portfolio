# GPU Interactive Compute — Engineering Implementation

This document covers GPU‑accelerated Jupyter and optional Spark RAPIDS integration.

## 1. Jupyter GPU Kernel

python -m ipykernel install \
  --prefix=/opt/jupyter/data \
  --name gpu-env \
  --display-name "Python (gpu-env)"

## 2. Validate GPU Kernel

import cupy as cp; cp.arange(10)
import torch; torch.cuda.is_available()
import cudf; cudf.Series([1,2,3])

## 3. Spark RAPIDS Accelerator (Optional)

wget https://repo1.maven.org/maven2/com/nvidia/rapids-4-spark_2.12/24.02.0/rapids-4-spark_2.12-24.02.0.jar

Spark config:

spark.plugins=com.nvidia.spark.SQLPlugin
spark.rapids.sql.enabled=true
spark.executor.resource.gpu.amount=1
spark.task.resource.gpu.amount=0.125

## 4. Validate Spark GPU

from pyspark.sql import SparkSession
spark = SparkSession.builder.getOrCreate()
df = spark.range(1_000_000)
df.selectExpr("sum(id)").show()
