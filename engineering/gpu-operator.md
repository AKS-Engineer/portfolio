# NVIDIA GPU Operator — Engineering Implementation

This document covers installation and validation of the NVIDIA GPU Operator for automated GPU lifecycle management.

## 1. Prerequisites

- Kubernetes cluster ready  
- containerd + CDI  
- NVIDIA drivers installed  

## 2. Install GPU Operator

kubectl create namespace gpu-operator
kubectl apply -f https://github.com/NVIDIA/gpu-operator/raw/v23.9.0/deployments/gpu-operator.yaml

## 3. Validate Operator Pods

kubectl get pods -n gpu-operator

Expected:
- driver
- toolkit
- device-plugin
- dcgm
- dcgm-exporter
- mig-manager (if supported)

## 4. Validate GPU Scheduling

kubectl run gpu-op-test \
  --image=nvidia/cuda:12.2.0-base \
  --command -- nvidia-smi

## 5. Validate DCGM Exporter

kubectl port-forward -n gpu-operator svc/dcgm-exporter 9400:9400
curl localhost:9400/metrics

## 6. MIG (If Supported)

nvidia-smi -mig 1
