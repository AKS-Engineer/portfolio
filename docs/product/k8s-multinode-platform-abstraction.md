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

## 4. Platform Abstractions Delivered

The Kubernetes environment was not designed as a collection of scripts or a cluster setup exercise. It was designed as a set of **platform abstractions** that hide infrastructure complexity and provide developers with predictable, cloud‑like behavior.

These abstractions define the boundaries between infrastructure, platform services, and workloads, ensuring that each layer can evolve independently while maintaining a consistent developer experience.

### 4.1 Networking Abstraction: Stable Identity for Every Node
Instead of exposing developers to DHCP variability or host‑file hacks, the platform provides:

- deterministic hostnames  
- stable IPs  
- consistent DNS resolution  

This abstraction ensures that workloads, kubeadm, registry access, and ingress routing behave predictably across rebuilds.

### 4.2 Image Distribution Abstraction: Local Registry as a Cloud Primitive
The private registry (`registry.example.internal:5000`) abstracts away:

- external image pulls  
- network variability  
- upstream rate limits  
- version drift  

Developers interact with a single, stable image source, mirroring the behavior of ECR, GCR, or ACR.

### 4.3 Package Availability Abstraction: Local APT Cache
The APT cache abstracts away:

- slow package downloads  
- upstream repository changes  
- dependency drift  

This turns provisioning into a fast, deterministic operation and ensures reproducibility across environments.

### 4.4 Ingress Abstraction: HAProxy as a Stable Entry Point
Instead of requiring `kubectl proxy` or ad‑hoc port forwarding, HAProxy provides:

- a single, stable ingress endpoint  
- predictable routing  
- a cloud‑like developer experience  

This abstraction hides Kubernetes networking complexity behind a consistent interface.

### 4.5 Cluster Lifecycle Abstraction: Rebuildable, Version‑Pinned Kubernetes
The combination of Vagrant, Ansible, and kubeadm creates a lifecycle abstraction:

- VMs are disposable  
- Kubernetes is reproducible  
- Versions are pinned  
- Configuration is codified  

Developers never interact with the underlying lifecycle mechanics — they get a stable cluster every time.

### 4.6 Optional Platform Layers: Modular Extensions Beyond the Core Cluster
The platform includes abstractions for enabling additional capabilities:

- Metrics Server  
- Kubernetes Dashboard  
- Istio service mesh  
- Ingress routing  
- Observability add‑ons  

These are intentionally modular: the cluster functions fully without them, but they can be enabled to support richer platform scenarios.

---

## 5. Key Design Decisions

### Stable Networking via DNS
A dedicated DNS server provides predictable hostnames and IPs, eliminating DHCP variability and ensuring consistent behavior across rebuilds.

### Local APT Cache
Package downloads are cached locally, improving rebuild speed and eliminating dependency on upstream repository availability.

### Private Container Registry
A local registry provides image locality, faster deployments, and consistent image versions.

### HAProxy as a Stable Ingress Layer
HAProxy abstracts away Kubernetes networking complexity and provides a single, stable ingress endpoint.

### Version Pinning
All critical components are pinned to ensure deterministic cluster rebuilds.

### Separation of Concerns
- Vagrant handles VM lifecycle  
- Ansible handles configuration  
- kubeadm handles cluster bootstrap  

---

## 6. Outcomes

### Developer Experience
- No manual port forwarding  
- No repeated image pulls  
- No environment drift  
- Stable endpoints for all workloads  
- Fast rebuilds  

### Platform Reliability
- Deterministic cluster creation  
- Version‑pinned components  
- Local caching for packages and images  
- Predictable networking  

### Extensibility
The platform supports:

- Kubernetes  
- Spark  
- Hadoop  
- Jenkins  
- Jupyter  
- Web applications  

All using the same underlying services.

### Cloud Alignment
The architecture mirrors cloud primitives:

| Cloud Concept | Platform Equivalent |
|---------------|---------------------|
| Region DNS | dns-server |
| Package mirrors | APT cache |
| Container registry | registry.example.internal |
| Load balancer | HAProxy |
| Control plane | cp-1 |
| Worker nodes | worker-1/2 |

---

## Summary
This project demonstrates how a local environment can be transformed into a small, self‑contained cloud by defining clear abstraction boundaries and building the supporting services that make distributed systems predictable. By combining deterministic networking, local caching, a private registry, HAProxy ingress, and reproducible Kubernetes automation, the platform delivers a stable, developer‑friendly experience that supports a wide range of workloads.

The result is a platform that abstracts away infrastructure complexity and provides a consistent foundation for experimentation, learning, and distributed systems development.
