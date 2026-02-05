# Engineering Implementation: Multi‑Node Kubernetes Platform (Vagrant + VMware + Ansible)

## Overview
This document describes the engineering implementation of a multi‑node Kubernetes platform built using Vagrant, VMware Workstation, Ansible, kubeadm, and a supporting services layer (DNS, APT cache, private registry, HAProxy). The environment behaves like a small, self‑contained cloud designed for Kubernetes, Spark, Hadoop, Jenkins, Jupyter, and general distributed systems experimentation.

All IP addresses and hostnames in this document use **example values** for illustration.

---

## 1. Architecture

### 1.1 Node Roles

**dns-server (external to this Vagrantfile)**  
- Authoritative DNS for the entire lab  
- Provides stable hostnames and IPs for all workloads  
- Acts as an APT cache server for fast, repeatable provisioning  
- Shared across Kubernetes, Spark, Hadoop, Jenkins, and other stacks  

**config-node**  
- Support node for Kubernetes and other platforms  
- Hosts:  
  - Private container registry (`registry.example.internal:5000`)  
  - HAProxy load balancer for Kubernetes ingress and service access  
  - TLS/CA material for internal trust  
  - IaC artifacts under `/vfiler`  
- Provides image locality and stable ingress endpoints for local development  

**cp-1**  
- Kubernetes control plane node  
- Runs kube-apiserver, scheduler, controller-manager, etcd  
- Bootstrapped with kubeadm using custom CIDRs and internal registry  

**worker-1, worker-2**  
- Kubernetes worker nodes  
- Run kubelet, kube-proxy, and user workloads  

### 1.2 Cluster Topology

| Node        | Example IP     | Role            | CPU | RAM   |
|-------------|----------------|------------------|-----|-------|
| cp-1        | 10.10.0.10     | Control Plane    | 2   | 16 GB |
| worker-1    | 10.10.0.11     | Worker           | 2   | 16 GB |
| worker-2    | 10.10.0.12     | Worker           | 2   | 16 GB |
| config-node | 10.10.0.20     | Registry + LB    | —   | —     |
| dns-server  | 10.10.0.30     | DNS + APT Cache  | —   | —     |

All nodes receive stable IPs and hostnames from `dns-server`.

---

## 2. Networking, DNS, and Supporting Services

### 2.1 Deterministic Networking
The DNS server provides:

- Forward and reverse DNS for all nodes  
- Stable hostnames (`cp-1.example.internal`, etc.)  
- Predictable IP allocation  

This eliminates DHCP variability and ensures kubeadm, registry, and HAProxy configurations remain stable across rebuilds.

### 2.2 APT Cache
The DNS server also acts as an APT cache, enabling:

- Fast provisioning  
- Reduced external dependencies  
- Reproducible builds even when upstream repos change  

### 2.3 Private Registry and HAProxy
`config-node` hosts:

- A private registry (`registry.example.internal:5000`) used by kubeadm and workloads  
- HAProxy to expose Kubernetes services without relying on `kubectl proxy`  
- CA certificates for internal TLS trust  

This creates a local ecosystem similar to cloud provider registries and load balancers.

---

## 3. Provisioning Flow

### 3.1 Vagrant Layer
The Vagrantfile:

- Uses VMware Workstation as the provider  
- Defines static MACs and IPs for all nodes  
- Mounts `/vfiler` from the host for IaC artifacts  
- Runs base OS updates  
- On `cp-1`, installs Ansible and triggers the cluster playbook  

Vagrant handles **VM lifecycle only**.  
All Kubernetes logic is delegated to Ansible.

### 3.2 Ansible Layer
The Ansible playbook executes in four stages:

1. **Container runtime + Kubernetes prep** on all Kubernetes nodes  
2. **Control plane initialization** on `cp-1`  
3. **Worker node join** on `worker-1` and `worker-2`  
4. **CNI and optional add‑ons** on `cp-1`  

---

## 4. Container Runtime and Kubernetes Prep

### 4.1 Kernel Modules and sysctl
All Kubernetes nodes enable:

- `overlay`  
- `br_netfilter`  

And configure:

```
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

### 4.2 containerd Installation and Configuration
Install containerd with pinned version:

```
containerd.io={{ containerd_version }}
```

Lock it:

```
apt-mark hold containerd.io
```

Copy custom config:

```
/vfiler/containerd/config.toml → /etc/containerd/
```

Restart containerd.

### 4.3 CA Distribution
To trust the internal registry:

- Copy `ca.example.internal.crt` to `/usr/local/share/ca-certificates`  
- Run `update-ca-certificates`  

### 4.4 Swap Disable
Kubernetes requires swap to be disabled:

- Remove swap entries from `/etc/fstab`  
- Run `swapoff -a`  

### 4.5 Kubernetes Components Installation
Install pinned versions of:

- kubeadm  
- kubelet  
- kubectl  
- kubernetes-cni  

Lock them:

```
apt-mark hold kubectl kubeadm kubelet kubernetes-cni
```

Restart kubelet.

---

## 5. Control Plane Initialization

`cp-1` runs:

```
kubeadm init \
  --apiserver-advertise-address=10.10.0.10 \
  --apiserver-cert-extra-sans=cp-1.example.internal \
  --node-name cp-1 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.244.5.0/24 \
  --image-repository registry.example.internal:5000 \
  --kubernetes-version {{ k8s_version }}
```

### 5.1 kubeconfig Distribution
Copy admin.conf to:

- `/home/vagrant/.kube/config`  
- `/vfiler/.kube/config`  

Copy cluster CA to:

- `/vfiler/certs/kube-cluster-ca.crt`  

### 5.2 Join Command Generation
Generate join command:

```
kubeadm token create --print-join-command
```

Store it in Ansible fact `join_command`.

---

## 6. Worker Node Join

Workers execute:

```
{{ hostvars['cp-1'].join_command }} >> node_joined.txt
```

This:

- Joins each worker to the cluster  
- Creates a marker file to avoid re‑joining on subsequent runs  

---

## 7. CNI and Optional Add‑ons

### 7.1 Calico CNI
Install Calico:

```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.29.1/manifests/calico.yaml
```

### 7.2 Optional Platform Layers
Commented sections include:

- Metrics Server  
- Kubernetes Dashboard  
- Istio (with Bookinfo demo)  
- Ingress routing  
- Add‑ons  

---

## 8. Lifecycle, Reproducibility, and Platform Behavior

### 8.1 Rebuild Workflow
Because:

- VMs are defined in Vagrant  
- IPs and hostnames are stable via DNS  
- Packages and images are cached locally  
- Versions are pinned  
- Configuration is codified in Ansible  

You can:

1. Destroy the VMs  
2. Recreate them via Vagrant  
3. Re-run the playbook  

…and the entire platform rebuilds **deterministically**.

### 8.2 Local “Mini Cloud” Behavior
With:

- DNS + APT cache  
- Private registry  
- HAProxy  
- Kubernetes  

You effectively have a **local cloud** capable of hosting:

- Spark  
- Hadoop  
- Jenkins  
- Jupyter  
- Web apps  
- Any containerized workload  

---

## Summary
This implementation builds a multi‑node Kubernetes platform that behaves like a small, self‑contained cloud. Vagrant and VMware manage VM lifecycle, Ansible codifies cluster configuration, kubeadm provides deterministic bootstrap, and supporting services (DNS, APT cache, private registry, HAProxy) create a fast, reproducible, developer‑friendly environment.

The result is not just a Kubernetes cluster, but a **platform foundation**: version‑pinned, locally self‑sufficient, and capable of supporting a wide range of distributed workloads.
