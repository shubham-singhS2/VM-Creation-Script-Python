# 🧪 Disk-Based Overlay VM Sandbox  
**KVM + Libvirt + Python | Ephemeral Virtual Environments**

This repository documents a **general-purpose, sandbox-grade virtual machine framework** built using **KVM, libvirt, QCOW2 disk overlays, and Python automation**.

The design enables **ephemeral virtual machines** that are created on demand, used for isolated workloads, and **fully destroyed after use**, ensuring **clean state, strong isolation, and reproducibility**.

This approach is suitable for **testing, research, experimentation, and security-sensitive workloads**.

---

## 🎯 Purpose of This Project

Traditional VM workflows often rely on:

- Full VM cloning (slow, disk-heavy)
- Snapshots (hard to automate, risky to reuse)

This project demonstrates a **clean, fast, and automation-friendly alternative** using **disk-based QCOW2 overlays**, similar to techniques used in **modern sandbox and CI environments**.

---

## 🧠 Core Design Concept

Instead of creating or restoring full VM disks:

1. Maintain **one clean base VM image**
2. Create a **QCOW2 overlay disk** per run
3. Boot the VM using the overlay
4. Perform tasks inside the VM
5. Destroy the VM and delete the overlay
6. Base image remains untouched

Each run is **stateless, disposable, and reproducible**.

---

## 🏗️ Architecture Overview

```
                          ┌────────────────────────┐
                          │  ubuntu-template.qcow2 │
                          │  (Read-only Base Image)│
                          └────────────┬───────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│ sandbox-vm-1.qcow2   │  │ sandbox-vm-2.qcow2   │  │ sandbox-vm-N.qcow2   │
│ (QCOW2 Overlay)      │  │ (QCOW2 Overlay)      │  │ (QCOW2 Overlay)      │
│ Ephemeral            │  │ Ephemeral            │  │ Ephemeral            │
└───────────┬──────────┘  └───────────┬──────────┘  └───────────┬──────────┘
            │                         │                         │
        Destroy VM               Destroy VM               Destroy VM
        Delete Overlay           Delete Overlay           Delete Overlay
```

---

## 🧪 Common Use Cases

- Secure software testing
- CI/CD isolated execution environments
- Configuration and OS testing
- Reverse engineering labs
- Security research and experimentation
- Training and learning environments
- Short-lived development sandboxes

---

## 🔧 Host Requirements

### Hardware
- CPU with Intel VT-x or AMD-V
- Minimum **8 GB RAM** (16 GB recommended)
- Minimum **20 GB free disk space**

### OS
- Ubuntu **20.04 / 22.04 / 24.04**

Verify virtualization support:
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

---

## 📦 Required Packages

```bash
sudo apt update
sudo apt install -y   qemu-kvm   libvirt-daemon-system   libvirt-clients   virt-manager   virt-viewer   bridge-utils   python3-libvirt   qemu-utils
```

Enable libvirt:
```bash
sudo systemctl enable --now libvirtd
```

Grant user permissions:
```bash
sudo usermod -aG libvirt $USER
newgrp libvirt
```

---

## 🖥️ Base Template VM Creation (One-Time)

### GUI Method (Recommended)

1. Launch `virt-manager`
2. Create a new VM using an Ubuntu ISO
3. Install OS normally
4. Apply system updates
5. Remove unnecessary packages
6. Shut down the VM

### Important Guidelines

- Treat the base image as **immutable**
- Never perform experiments directly on the base VM
- Use overlays for all execution

---

## 💾 Convert Disk to Template

Locate VM disk:
```bash
ls /var/lib/libvirt/images/
```

Example:
```text
ubuntu-template.qcow2
```

This file becomes your **golden base image**.

---

## 🔐 Template Hardening (Recommended)

- Remove SSH keys
- Disable cloud-init if unused
- Clear logs and history
- Disable auto-updates (optional)
- Verify clean boot state

---

## 🚀 Sandbox Automation with Python

### File: `sandbox_vm.py`

```python
#!/usr/bin/env python3
import libvirt
import subprocess
import os
import sys

TEMPLATE = "/var/lib/libvirt/images/ubuntu-template.qcow2"
OVERLAY  = "/var/lib/libvirt/images/sandbox-vm.qcow2"
VM_NAME  = "sandbox-vm"
RAM_MB   = 4096
VCPUS    = 2
DISK_SIZE_GB = 25

def create_overlay():
    if os.path.exists(OVERLAY):
        os.remove(OVERLAY)
    subprocess.run([
        "qemu-img", "create", "-f", "qcow2",
        "-F", "qcow2", "-b", TEMPLATE,
        OVERLAY, f"{DISK_SIZE_GB}G"
    ], check=True)

def generate_domain_xml():
    return f"""
<domain type='kvm'>
  <name>{VM_NAME}</name>
  <memory unit='MiB'>{RAM_MB}</memory>
  <vcpu>{VCPUS}</vcpu>
  <os>
    <type arch='x86_64' machine='pc'>hvm</type>
  </os>
  <devices>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='{OVERLAY}'/>
      <target dev='vda' bus='virtio'/>
    </disk>
    <interface type='network'>
      <source network='default'/>
      <model type='virtio'/>
    </interface>
    <graphics type='vnc' port='-1'/>
  </devices>
</domain>
"""

def main():
    create_overlay()
    conn = libvirt.open("qemu:///system")
    if conn is None:
        sys.exit("Failed to connect to libvirt")

    domain = conn.defineXML(generate_domain_xml())
    domain.create()
    print("Sandbox VM started.")
    input("Press ENTER when finished...")

    if domain.isActive():
        domain.destroy()
    domain.undefine()
    os.remove(OVERLAY)
    print("Sandbox cleaned successfully.")

if __name__ == "__main__":
    main()
```

---

## ▶️ Running the Sandbox

```bash
python3 sandbox_vm.py
```

### Execution Flow

1. Overlay disk is created
2. VM boots from overlay
3. Tasks are executed inside the VM
4. User completes work
5. VM is destroyed
6. Overlay disk is deleted
7. System returns to clean state

---

## 🔄 Overlay vs Snapshot Comparison

| Feature | Snapshot | Overlay |
|-------|---------|--------|
| Parallel VMs | ❌ | ✅ |
| Automation Friendly | ❌ | ✅ |
| Performance | Medium | Fast |
| Cleanup | Manual | Automatic |
| Reusability Risk | High | None |

---

## 🔐 Security & Isolation Considerations

- Treat each VM as disposable
- Never reuse overlay disks
- Use NAT or isolated networks
- Avoid shared folders
- Disable clipboard & USB passthrough if needed
- Keep host services isolated

---

## 🧩 Future Enhancements

- RAM-backed overlay disks
- Parallel VM orchestration
- Windows-based sandbox templates
- Network traffic capture
- Snapshot export for debugging
- Integration with CI pipelines

---

## 🏁 Conclusion

This project demonstrates:

✔ Clean and repeatable VM sandboxing  
✔ Strong isolation using disk overlays  
✔ Automation-ready architecture  
✔ Efficient resource usage  

It is suitable for **testing, experimentation, CI systems, and secure execution environments**.

---
