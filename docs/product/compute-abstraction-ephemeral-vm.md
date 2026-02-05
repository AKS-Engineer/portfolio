# PM Case Study: Designing a Compute Abstraction for Ephemeral VM Lifecycles

## Overview
This case study describes how I framed and designed a **compute abstraction** for environments where virtual machines are frequently destroyed, recreated, or reconfigured. The goal was to make the underlying VM lifecycle **irrelevant** to the user experience, while preserving reliability, state, and predictability for platform consumers.

The work builds on patterns validated in my lab (Vagrant + VMware, Jenkins persistence) and generalizes them into a reusable abstraction that can be applied to other services and platforms.

---

## 1. Problem Context

In many lab and platform environments, VMs are:

- frequently rebuilt  
- used as disposable test surfaces  
- changed via automation, scripts, or configuration experiments  

However, the workloads running on those VMs—CI/CD systems, artifact repositories, internal tools—often **assume stability**:

- they expect state to persist  
- they assume the host is long‑lived  
- they break when the VM is destroyed  

This creates a mismatch:

- **Infrastructure reality:** compute is ephemeral  
- **User expectation:** services are stable  

The core question:

> How do we design a compute abstraction that embraces ephemeral VMs while still delivering a stable platform experience?

---

## 2. Users and Stakeholders

### Developer
- Wants a stable endpoint (URL, API, UI)  
- Expects state and history to persist  
- Does not want to care about VM lifecycles  

### Platform Engineer
- Needs freedom to rebuild, reconfigure, and experiment  
- Wants clear patterns for attaching/detaching state  
- Needs predictable recovery and minimal manual steps  

### SRE / Reliability Owner
- Wants clear failure domains  
- Needs repeatable recovery workflows  
- Cares about drift, backups, and incident blast radius  

---

## 3. Product Goal

Design a **compute abstraction** that:

- treats VMs as disposable  
- treats state as durable  
- provides a stable experience to users  
- gives platform engineers operational flexibility  
- can be reused across multiple services  

In other words:

> “You can destroy the VM as often as you want. The service experience remains stable.”

---

## 4. Abstraction Definition

The abstraction is defined as a **contract** between compute and state:

- **Compute:**  
  - ephemeral  
  - replaceable  
  - stateless by design  
  - safe to destroy and recreate  

- **State:**  
  - durable  
  - independently managed  
  - attached to compute at runtime  
  - survives across lifecycles  

This mirrors cloud-native patterns:

- EC2 + EBS  
- GCP VM + Persistent Disk  
- Azure VM + Data Disk  

The key is not just the technology, but the **product boundary**:

> “Compute can be replaced. State cannot be casually lost.”

---

## 5. Requirements

### Functional
- Services must retain state across VM rebuilds  
- Compute nodes must be replaceable without manual data recovery  
- The pattern must be reusable across multiple workloads  

### Non‑Functional
- Predictable attach/mount behavior  
- Clear operational runbooks  
- Minimal friction for platform engineers  
- Alignment with cloud-native patterns for future migration  

---

## 6. Model: Ephemeral Compute, Durable State

The compute abstraction is expressed as a simple model:

1. **Compute Node**
   - Runs the service (e.g., Jenkins, GitLab, Nexus)  
   - Can be destroyed and recreated at any time  

2. **State Volume**
   - Holds service data (config, jobs, artifacts, metadata)  
   - Lives outside the compute lifecycle  
   - Is attached/mounted at runtime  

3. **Binding Logic**
   - On boot, compute attaches/mounts the state volume  
   - The service reads/writes only to that volume  
   - On destroy, the volume is detached but not deleted  

This model is intentionally generic so it can be applied to multiple services.

---

## 7. Example: Applying the Abstraction to Jenkins

In the Jenkins case:

- **Compute:** Vagrant VM with Jenkins installed  
- **State:** VMware VMDK mounted at `/jenkins-data`  
- **Binding:** `JENKINS_HOME` points to `/jenkins-data`  

Outcome:

- VM can be destroyed and recreated  
- Jenkins resumes with full history, plugins, and credentials  
- The user experience is stable, even if the VM is not  

This validates the abstraction in a concrete scenario.

---

## 8. Tradeoffs and Decisions

### Tradeoffs
- **Manual vs automated attach/detach:**  
  - Manual steps are explicit and reliable, but add operational overhead  
  - Automation reduces friction but increases complexity and failure modes  

- **Single‑service vs shared patterns:**  
  - Service‑specific tuning vs generic platform pattern  
  - Chosen approach: start with service‑specific, then generalize  

- **Local lab vs cloud alignment:**  
  - Lab constraints (Vagrant + VMware) vs cloud-native APIs  
  - Chosen approach: design patterns that map cleanly to cloud concepts later  

### Key Decisions
- Treat the abstraction as a **platform contract**, not just a script  
- Optimize for clarity and repeatability over cleverness  
- Design with future automation in mind, but validate manually first  

---

## 9. Outcomes

The compute abstraction delivered:

- **Operational flexibility:**  
  Platform engineers can rebuild VMs without breaking services.

- **Reliability:**  
  State is preserved across destructive operations.

- **Reusability:**  
  The same pattern can be applied to other stateful services.

- **Cloud readiness:**  
  The model maps directly to managed disks and volumes in major clouds.

It also created a shared language:

- “Compute is ephemeral.”  
- “State is durable.”  
- “The binding is the contract.”

---

## 10. How This Scales

This abstraction can be extended to:

- **Multiple services:** Jenkins, GitLab, Nexus, internal tools  
- **Different environments:** local lab, on‑prem, cloud  
- **Higher‑level constructs:**  
  - compute classes  
  - storage tiers  
  - environment profiles  
  - platform templates  

Over time, this becomes part of the **platform vocabulary**:

> “This service runs on ephemeral compute with a durable state volume.”

---

## Summary
By explicitly separating compute from state and treating that separation as a product boundary, this compute abstraction turns fragile, VM‑bound services into stable, repeatable platform experiences. It gives platform engineers the freedom to rebuild and experiment, while giving developers a consistent, reliable surface to work against. The pattern is simple, extensible, and aligns naturally with cloud-native infrastructure models.
