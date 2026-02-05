# Engineering Implementation: Multi‑Node Kubernetes Platform (Vagrant + VMware + Ansible)

## Overview
This document describes the engineering implementation of a multi‑node Kubernetes platform built using Vagrant, VMware Workstation, Ansible, kubeadm, and a supporting services layer (DNS, APT cache, private registry, HAProxy). The environment behaves like a small, self‑contained cloud designed for Kubernetes, Spark, Hadoop, Jenkins, Jupyter, and general distributed systems experimentation.

The design emphasizes reproducibility, deterministic networking, version pinning, and local self‑sufficiency.

---

## 1. Architecture

### 1.1 Node Roles

**dns-server (external to this Vagrantfile)**  
- Authoritative DNS for the entire lab  
- Provides stable hostnames and IPs for all workloads  
- Acts as an APT cache server for fast, repeatable provisioning  
- Shared across Kubernetes, Spark, Hadoop, Jenkins, and other stacks  

**k8s-config**  
- Support node for Kubernetes and other platforms  
- Hosts:  
  - Private Docker/container registry (`k8s-config.mylab.io:5000`)  
  - HAProxy load balancer for Kubernetes ingress and service access  
  - TLS/CA material for internal trust  
  - IaC artifacts under `/vfiler`  
- Provides image locality and stable ingress endpoints for local development  

**k8s-master**  
- Kubernetes control plane node  
- Runs kube-apiserver, scheduler, controller-manager, etcd  
- Bootstrapped with kubeadm using custom CIDRs and internal registry  

**k8s-node01, k8s-node02**  
- Worker nodes  
- Run kubelet, kube-proxy, and user workloads  

### 1.2 Cluster Topology

| Node        | IP Address       | Role            | CPU | RAM   |
|-------------|------------------|------------------|-----|-------|
| k8s-master  | 192.168.44.12    | Control Plane    | 2   | 16 GB |
| k8s-node01  | 192.168.44.13    | Worker           | 2   | 16 GB |
| k8s-node02  | 192.168.44.14    | Worker           | 2   | 16 GB |
| k8s-config  | 192.168.44.11    | Registry + LB    | —   | —     |
| dns-server  | 192.168.44.x     | DNS + APT Cache  | —   | —     |

All nodes receive stable IPs and hostnames from `dns-server`.

---

## 2. Networking, DNS, and Supporting Services

### 2.1 Deterministic Networking
The DNS server provides:

- Forward and reverse DNS for all nodes  
- Stable hostnames (`k8s-master.mylab.io`, etc.)  
- Predictable IP allocation  

This eliminates DHCP variability and ensures kubeadm, registry, and HAProxy configurations remain stable across rebuilds.

### 2.2 APT Cache
The DNS server also acts as an APT cache, enabling:

- Fast provisioning  
- Reduced external dependencies  
- Reproducible builds even when upstream repos change  

### 2.3 Private Registry and HAProxy
`k8s-config` hosts:

- A private registry (`k8s-config.mylab.io:5000`) used by kubeadm and workloads  
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
- On `k8s-master`, installs Ansible and triggers the cluster playbook  

Vagrant handles **VM lifecycle only**.  
All Kubernetes logic is delegated to Ansible.

### 3.2 Ansible Layer
The Ansible playbook (`k8s-cluster-bun-2004.yaml`) executes in four stages:

1. **Container runtime + Kubernetes prep** on all Kubernetes nodes  
2. **Control plane initialization** on `k8s-master`  
3. **Worker node join** on `k8s-node01` and `k8s-node02`  
4. **CNI and optional add‑ons** on `k8s-master`  

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

These settings ensure proper packet forwarding and iptables visibility.

### 4.2 containerd Installation and Configuration
Install containerd with pinned version:

```
containerd.io={{ containerd_version }}
```

Then lock it:

```
apt-mark hold containerd.io
```

Copy custom config:

```
/vfiler/containerd/config.toml → /etc/containerd/
```

This typically includes:

- cgroup driver configuration  
- registry mirrors (including the internal registry)  
- runtime settings  

Restart containerd after configuration.

### 4.3 CA Distribution
To trust the internal registry:

- Copy `ca.mylab.io.crt` to `/usr/local/share/ca-certificates`  
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

Then lock them:

```
apt-mark hold kubectl kubeadm kubelet kubernetes-cni
```

Restart kubelet.

---

## 5. Control Plane Initialization

`k8s-master` runs:

```
kubeadm init \
  --apiserver-advertise-address={{ ansible_host_public_ip }} \
  --apiserver-cert-extra-sans={{ ansible_host }} \
  --node-name k8s-master \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.244.5.0/24 \
  --image-repository k8s-config.mylab.io:5000 \
  --kubernetes-version {{ k8s_version }}
```

Key design choices:

- Stable advertise address  
- SANs for TLS correctness  
- Custom Pod and Service CIDRs  
- Internal registry for image pulls  
- Version‑pinned Kubernetes  

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
{{ hostvars['k8s-master'].join_command }} >> node_joined.txt
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

When enabled, these turn the cluster into a full platform with:

- Observability  
- Web UI  
- Service mesh  
- Ingress routing  

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

All with:

- Stable endpoints  
- Local image caching  
- Fast rebuilds  
- Predictable behavior  

---

## Summary
This implementation builds a multi‑node Kubernetes platform that behaves like a small, self‑contained cloud. Vagrant and VMware manage VM lifecycle, Ansible codifies cluster configuration, kubeadm provides deterministic bootstrap, and supporting services (DNS, APT cache, private registry, HAProxy) create a fast, reproducible, developer‑friendly environment.

The result is not just a Kubernetes cluster, but a **platform foundation**: version‑pinned, locally self‑sufficient, and capable of supporting a wide range of distributed workloads.
