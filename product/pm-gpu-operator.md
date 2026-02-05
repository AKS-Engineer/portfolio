# Product Spec: NVIDIA GPU Operator  
Platform Abstraction Document

---

## 1. Purpose

The GPU Operator automates the **entire GPU lifecycle** inside Kubernetes:

- drivers  
- toolkit  
- container runtime  
- device plugin  
- DCGM  
- MIG  

It is the automation backbone of the GPU platform.

---

## 2. Problem / Context

Manual GPU lifecycle management is error‑prone:

- driver upgrades break workloads  
- toolkit mismatches cause runtime failures  
- device plugin versions drift  
- DCGM is often misconfigured  

Cloud providers solve this with internal automation.  
The GPU Operator provides that automation for Kubernetes.

---

## 3. Platform Abstraction

**The platform guarantees:**

- Automated GPU driver lifecycle  
- Automated toolkit lifecycle  
- Automated device plugin deployment  
- Automated DCGM + exporter  
- Automated MIG configuration  

**Users do not manage:**

- drivers  
- toolkit  
- runtime configuration  
- device plugin  
- DCGM  

---

## 4. Conceptual Architecture

```
GPU Operator
        │
        ├── Driver Manager
        ├── Toolkit Manager
        ├── Device Plugin
        ├── DCGM + Exporter
        └── MIG Manager
```

---

## 5. User Experience

Developers experience:

- GPU pods that always work  
- No driver/toolkit concerns  
- GPU metrics available by default  

Platform engineers experience:

- Declarative GPU lifecycle  
- Clear operational boundaries  
- Reduced node drift  

---

## 6. Responsibilities & Boundaries

### Platform Responsibilities
- Deploy and maintain Operator  
- Validate Operator version compatibility  
- Monitor Operator health  

### Developer Responsibilities
- Use supported GPU images  
- Report GPU scheduling issues  

---

## 7. Non‑Goals

- Not a workload scheduler  
- Not a GPU quota system  
- Not a multi‑tenant isolation layer  

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Driver conflicts | Operator‑owned lifecycle |
| Toolkit mismatch | Version pinning |
| DCGM failures | Exporter health checks |

---

## 9. Future Extensions

- Multi‑tenant GPU isolation  
- GPU admission policies  
- GPU node auto‑repair  

---
