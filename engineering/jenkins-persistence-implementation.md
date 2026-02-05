# Engineering Implementation: Jenkins Persistence on Ephemeral VMs (Vagrant + VMware)

## Overview
This document describes the engineering implementation behind separating Jenkins state from an ephemeral Vagrant VM running on the VMware provider. The goal was to ensure that Jenkins configuration, job history, credentials, and plugin data survive VM destruction and recreation.

The solution mirrors cloud-native patterns such as EC2 + EBS, GCP VM + PD, and Azure VM + Data Disk.

---

## 1. Architecture

### Components
- **Vagrant VM (ephemeral)**  
  - Ubuntu base image  
  - Jenkins service  
  - Networking and runtime dependencies  

- **Persistent VMware VMDK (durable)**  
  - Lives outside Vagrant’s lifecycle  
  - Stores Jenkins HOME  
  - Attached/detached manually  

### High-Level Diagram

```
+-------------------------+       +---------------------------+
|     Vagrant VM          |       |   Persistent VMDK Disk    |
|-------------------------|       |---------------------------|
| OS (Ubuntu)             |       | ext4 filesystem           |
| Jenkins runtime         | <---> | /jenkins-data (JENKINS_HOME)
| Plugins (runtime)       |       | Job history, credentials  |
| Ephemeral filesystem    |       | Plugin state, workspaces  |
+-------------------------+       +---------------------------+
```

---

## 2. Disk Creation

A dedicated VMDK was created manually using VMware tooling:

```
vmware-vdiskmanager -c -t 0 -s 20GB jenkins-data.vmdk
```

Key decisions:
- **Type 0 (monolithicSparse)** for portability  
- Disk stored **outside** the Vagrant project directory  
- Ensures Vagrant destroy/up does not affect the disk  

---

## 3. Attaching the Disk to the VM

In the Vagrantfile:

```ruby
config.vm.provider "vmware_desktop" do |v|
  v.vmx["scsi0:1.present"] = "TRUE"
  v.vmx["scsi0:1.fileName"] = "/absolute/path/to/jenkins-data.vmdk"
end
```

Notes:
- VMware provider does **not** support Vagrant-managed disks  
- Manual attachment ensures lifecycle separation  
- Absolute paths required for VMware  

---

## 4. Formatting and Mounting the Disk

Once the VM boots:

### Identify the disk
```
lsblk
```

Assume it appears as `/dev/sdb`.

### Format as ext4
```
sudo mkfs.ext4 /dev/sdb
```

### Create mount point
```
sudo mkdir /jenkins-data
```

### Mount manually for first-time setup
```
sudo mount /dev/sdb /jenkins-data
```

### Add to `/etc/fstab` for predictable behavior
```
/dev/sdb    /jenkins-data    ext4    defaults    0 0
```

---

## 5. Migrating Jenkins HOME

Stop Jenkins:
```
sudo systemctl stop jenkins
```

Move existing data:
```
sudo rsync -av /var/lib/jenkins/ /jenkins-data/
```

Update Jenkins configuration:

```
sudo sed -i 's|JENKINS_HOME=.*|JENKINS_HOME=/jenkins-data|' /etc/default/jenkins
```

Fix ownership:
```
sudo chown -R jenkins:jenkins /jenkins-data
```

Start Jenkins:
```
sudo systemctl start jenkins
```

Validation:
- UI loads  
- Jobs appear  
- Credentials intact  
- Plugins functional  

---

## 6. Lifecycle Workflow

### Before destroying the VM
Detach the disk in VMware UI or remove the Vagrantfile attachment temporarily.

### After recreating the VM
Reattach the disk and boot.

### Jenkins resumes exactly where it left off
- No job loss  
- No plugin reinstallation  
- No credential resets  
- No drift  

This creates a **deterministic recovery path**.

---

## 7. Why This Pattern Works

### Reliability
State is isolated from compute, eliminating drift and rebuild failures.

### Reproducibility
VMs can be destroyed and recreated without affecting Jenkins.

### Cloud Alignment
Matches patterns used by AWS, GCP, Azure for stateful workloads.

### Extensibility
The same pattern can be applied to:
- GitLab  
- Nexus  
- SonarQube  
- Any stateful service  

---

## 8. Validation Checklist

- Disk mounts automatically on boot  
- Jenkins HOME points to `/jenkins-data`  
- Permissions correct (`jenkins:jenkins`)  
- Jobs persist across VM rebuilds  
- Plugin state persists  
- Credentials persist  
- No drift after multiple destroy/up cycles  

---

## Summary
This implementation creates a clean separation between ephemeral compute and persistent state, enabling Jenkins to operate reliably in an environment where VMs are frequently rebuilt. The pattern is simple, repeatable, and aligns with cloud-native best practices for stateful workloads.
