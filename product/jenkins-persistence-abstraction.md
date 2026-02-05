# PM Case Study: Designing a Reliability Abstraction for Jenkins in an Ephemeral Compute Environment

## Overview
This case study demonstrates how I defined an abstraction boundary between ephemeral compute and persistent state to improve CI/CD reliability in a constrained lab environment. The goal was to create a developer experience where Jenkins behaves predictably even when the underlying VM is frequently destroyed, rebuilt, or reconfigured.

This work reflects how I approach platform product design, reliability as a feature, and developer experience simplification.

---

## 1. Problem Statement
Engineers using Jenkins often experience:

- fragile build environments  
- inconsistent state after VM rebuilds  
- loss of job history, credentials, and plugins  
- unpredictable failures tied to underlying infrastructure changes  

In my lab, Vagrant + VMware workflows made VM destruction common. Jenkins, however, is not designed to survive ephemeral compute without explicit state separation.

**Core problem:**  
How do we make Jenkins reliable when the compute layer is disposable?

---

## 2. User Personas

### Developer
- Wants stable CI/CD pipelines  
- Expects job history and credentials to persist  
- Doesn’t want to understand infrastructure details  

### Platform Engineer
- Needs to rebuild VMs frequently  
- Wants predictable recovery  
- Needs a clean separation between compute and state  

### SRE / Reliability Owner
- Wants clear failure domains  
- Needs a repeatable recovery workflow  
- Wants to eliminate configuration drift  

---

## 3. Product Requirements

### Functional
- Jenkins state must persist across VM destruction  
- Compute layer must remain fully ephemeral  
- Recovery must be deterministic and low‑friction  
- No reliance on shared folders with permission issues  

### Non‑Functional
- Secure filesystem  
- Predictable mount behavior  
- Minimal operational overhead  
- Clear lifecycle boundaries  

---

## 4. Abstraction Design: Ephemeral Compute + Persistent State
I defined a simple but powerful abstraction boundary:

### Compute Layer (Ephemeral)
- Vagrant VM  
- OS  
- Jenkins service  
- Networking  
- Plugins installed at runtime  

### State Layer (Persistent)
- Jenkins HOME  
- Job history  
- Credentials  
- Workspaces  
- Plugin data  

This mirrors cloud patterns (EC2 + EBS, GCP VM + PD, Azure VM + Data Disk).

**Abstraction:**  
The VM can be destroyed at any time. Jenkins state cannot.

---

## 5. Implementation (Engineering Input to Product Decision)

### Persistent Disk
- Created a dedicated VMware VMDK  
- Stored outside the Vagrant lifecycle  
- Attached/detached manually to enforce lifecycle separation  

### Filesystem
- Formatted as ext4  
- Mounted at `/jenkins-data`  
- Added to `/etc/fstab` for predictable behavior  

### Jenkins Integration
- Migrated Jenkins HOME to persistent disk  
- Ensured correct ownership and permissions  
- Validated plugin and job history persistence  

### Lifecycle Workflow
- Before destroy: detach disk  
- After up: reattach disk  
- Jenkins resumes exactly where it left off  

This workflow becomes a platform guarantee.

---

## 6. Tradeoffs Considered

### Rejected Options
- HGFS shared folders (permission issues, unreliable for Jenkins)  
- Vagrant-managed disks (unsupported by VMware provider)  
- Storing Jenkins state inside the VM (breaks reliability)  

### Accepted Tradeoffs
- Manual disk attach/detach (explicit but reliable)  
- Slight operational overhead in exchange for deterministic behavior  

---

## 7. Outcome
The abstraction delivered:

- 100% Jenkins state persistence across VM rebuilds  
- Zero configuration drift  
- Predictable recovery  
- Clear separation of concerns  
- A reusable pattern for other stateful services  

This became a foundational building block for future platform abstractions such as:

- compute classes  
- storage tiers  
- accelerator profiles  
- developer workflows  

---

## 8. What This Demonstrates (PM Competencies)

### Platform Thinking
Defined a clean boundary between compute and state.

### Abstraction Design
Simplified a complex reliability problem into a reusable model.

### Developer Experience
Reduced friction and eliminated fragile workflows.

### Tradeoff Analysis
Balanced operational overhead with reliability guarantees.

### Product Mindset
Focused on outcomes, not implementation details.

---

## Summary
This project shows how I think as a Platform Product Manager:

- identify user pain  
- define abstraction boundaries  
- design workflows that reduce friction  
- create reliability guarantees  
- translate engineering complexity into intuitive models  

The engineering work is real.  
The product thinking is intentional.  
The narrative is credible for PM roles in AI infrastructure, compute platforms, and developer experience.
