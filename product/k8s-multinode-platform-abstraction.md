# PM Case Study: Designing a Multi‑Node Kubernetes Platform for Local Cloud Development

## Overview
This case study describes how I designed a multi‑node Kubernetes platform that behaves like a small, self‑contained cloud. The goal was to create a reproducible, stable environment for running distributed systems (Kubernetes, Spark, Hadoop, Jenkins, Jupyter, web apps) without relying on external infrastructure or unpredictable local machine setups.

The platform abstracts away VM lifecycles, networking complexity, image distribution, and ingress management so developers can interact with a consistent, predictable environment.

---

## 1. Problem Context

Local development environments for distributed systems often suffer from:

- inconsistent networking  
- unstable IPs and hostnames  
- slow or unreliable package downloads  
- repeated image pulls from the internet  
- manual port forwarding or `kubectl proxy` usage  
- clusters that break when rebuilt  
- lack of reproducibility across machines  

These issues create friction for developers and make it difficult to experiment with real platform behaviors.

The challenge was to design a **local platform abstraction** that:

- behaves like a cloud  
- rebuilds deterministically  
- isolates complexity behind clear boundaries  
- supports multiple workloads beyond Kubernetes  

---

## 2. Users and Their Needs

### Developer
- Wants a stable cluster endpoint  
- Expects workloads to deploy consistently  
- Doesn’t want to manage networking, ingress, or image pulls  
- Needs fast rebuilds and predictable behavior  

### Platform Engineer
- Needs deterministic networking  
- Wants to rebuild clusters without drift  
- Needs a local registry for image locality  
- Wants automation that is repeatable and version‑pinned  

### Data/ML Engineer
- Needs a consistent environment for Spark, Hadoop, Jupyter  
- Wants to avoid dependency conflicts and environment drift  

---

## 3. Product Goal

Design a **local cloud abstraction** that provides:

- stable networking and DNS  
- local package caching  
- local container registry  
- load‑balanced ingress  
- reproducible Kubernetes clusters  
- fast rebuilds  
- support for multiple distributed workloads  

The platform should feel like a small cloud region:

> “You can destroy and rebuild everything, and the environment comes back exactly the same.”

---

## 4. Abstraction Boundaries

The platform is defined by three clear boundaries:

### **1. Infrastructure Layer (Vagrant + VMware)**
- VMs are ephemeral  
- IPs and hostnames are stable  
- VM lifecycle is fully automated  

### **2. Platform Services Layer (dns-server + k8s-config)**
- DNS  
- APT cache  
- Private registry  
- HAProxy ingress  
- CA distribution  

These services provide the foundation for all workloads.

### **3. Kubernetes Layer (kubeadm + Ansible)**
- Deterministic control plane bootstrap  
- Version‑pinned components  
- CNI networking  
- Optional platform add‑ons (Dashboard, Metrics Server, Istio)  

This separation ensures that each layer can evolve independently.

---

## 5. Key Design Decisions

### **Stable Networking via DNS**
Instead of relying on DHCP or host‑file hacks, the platform uses a dedicated DNS server:

- predictable hostnames  
- stable IPs  
- consistent kubeadm configuration  
- reliable registry and ingress endpoints  

This eliminates a major source of drift.

### **Local APT Cache**
Package downloads are cached locally:

- faster provisioning  
- reproducible builds  
- independence from upstream repo changes  

This improves rebuild speed and reliability.

### **Private Container Registry**
A local registry provides:

- image locality  
- faster deployments  
- offline‑friendly behavior  
- consistent image versions  

Kubeadm and containerd are configured to use this registry by default.

### **HAProxy as a Stable Ingress Layer**
Instead of relying on `kubectl proxy`, HAProxy provides:

- a single, stable ingress endpoint  
- predictable routing  
- a cloud‑like developer experience  

This abstracts away Kubernetes networking complexity.

### **Version Pinning**
All critical components are pinned:

- containerd  
- kubeadm  
- kubelet  
- kubectl  
- CNI  
- CoreDNS  
- etcd  

This ensures deterministic cluster rebuilds.

### **Separation of Concerns**
- Vagrant handles VM lifecycle  
- Ansible handles configuration  
- kubeadm handles cluster bootstrap  

This creates a clean, maintainable architecture.

---

## 6. Platform Model

The platform follows a simple model:

### **1. Compute is disposable**
VMs can be destroyed and recreated at any time.

### **2. Networking and services are stable**
DNS, registry, and HAProxy provide consistent endpoints.

### **3. Kubernetes is reproducible**
Version pinning and IaC ensure deterministic rebuilds.

### **4. Workloads are portable**
Spark, Hadoop, Jenkins, Jupyter, and web apps all run on the same foundation.

This model mirrors cloud provider design patterns.

---

## 7. Outcomes

### **Developer Experience**
- No manual port forwarding  
- No repeated image pulls  
- No environment drift  
- Stable endpoints for all workloads  
- Fast rebuilds  

### **Platform Reliability**
- Deterministic cluster creation  
- Version‑pinned components  
- Local caching for packages and images  
- Predictable networking  

### **Extensibility**
The platform supports:

- Kubernetes  
- Spark  
- Hadoop  
- Jenkins  
- Jupyter  
- Web applications  

All using the same underlying services.

### **Cloud Alignment**
The architecture mirrors cloud primitives:

| Cloud Concept | Platform Equivalent |
|---------------|---------------------|
| Region DNS | dns-server |
| Package mirrors | APT cache |
| Container registry | k8s-config registry |
| Load balancer | HAProxy |
| Control plane | k8s-master |
| Worker nodes | k8s-node01/02 |

This makes the platform a strong foundation for learning, experimentation, and platform PM thinking.

---

## Summary
This project demonstrates how a local environment can be transformed into a small, self‑contained cloud by defining clear abstraction boundaries and building the supporting services that make distributed systems predictable. By combining deterministic networking, local caching, a private registry, HAProxy ingress, and reproducible Kubernetes automation, the platform delivers a stable, developer‑friendly experience that supports a wide range of workloads.

The result is a platform that abstracts away infrastructure complexity and provides a consistent foundation for experimentation, learning, and distributed systems development.
