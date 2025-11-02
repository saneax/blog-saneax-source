---
title: "Creating and Customizing Virtual Machines with NUMA Awareness on Modern Linux (Fedora/Ubuntu 2025)"
date: 2025-11-02T10:00:00+05:30
description: "A step-by-step guide to creating virtual machines on Fedora or Ubuntu using QEMU/KVM and libvirt, including NUMA-aware CPU and memory configuration for optimized performance on modern hardware."
summary: "Learn how to create, manage, and tune virtual machines with NUMA awareness using QEMU, KVM, and libvirt on Fedora and Ubuntu. Includes examples for NUMA topology, CPU pinning, hugepages, and bridge networking."
slug: "virtualization-numa-guide"
draft: false
authors: ["Sanjay Upadhyay"]
tags:
  - Virtualization
  - KVM
  - libvirt
  - NUMA
  - Fedora
  - Ubuntu
  - QEMU
  - Linux
  - Performance
categories:
  - Infrastructure
  - Linux Administration
cover:
  image: "/images/posts/virtualization-numa-guide-cover.jpg"
  alt: "NUMA-aware virtual machine topology visualization on Linux"
  caption: "NUMA-aware VM configuration using libvirt on Fedora and Ubuntu"
  relative: true
keywords:
  - KVM NUMA tutorial
  - libvirt NUMA configuration
  - Fedora virtualization
  - Ubuntu virtualization
  - NUMA-aware virtual machine setup
readingTime: true
showToc: true
tocOpen: true
coverCaption: true
related:
  threshold: 0.5
  includeNewer: true
  toLower: false
---

## Title

**“Creating and Customizing Virtual Machines with NUMA Awareness on Modern Linux (Fedora/Ubuntu 2025)”**

---

## Introduction

Virtualization has evolved significantly, and tools like **libvirt**, **virt-manager**, and **QEMU/KVM** now allow fine-grained control over CPU topology, NUMA nodes, and memory allocation.
In this guide, we’ll create a **fully functional VM** using both **GUI** and **CLI** methods, compatible with **Fedora 42** and **Ubuntu 24.04 LTS**.
We’ll also explore how to assign **NUMA nodes**, distribute **vCPUs and memory**, and customize devices and networks.

---

##  1. Install Dependencies

### Fedora

```bash
sudo dnf install -y @virtualization libvirt virt-install virt-manager qemu-kvm
sudo systemctl enable --now libvirtd
```

### Ubuntu

```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients virt-manager bridge-utils
sudo systemctl enable --now libvirtd
```

Check KVM support:

```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```

---

##  2. Create a Storage Pool and Network

### Create a storage pool

```bash
sudo mkdir -p /var/lib/libvirt/images
sudo virsh pool-define-as default dir - - - - "/var/lib/libvirt/images"
sudo virsh pool-autostart default
sudo virsh pool-start default
```

### Verify default NAT network

```bash
sudo virsh net-list --all
```

If missing:

```bash
sudo virsh net-define /usr/share/libvirt/networks/default.xml
sudo virsh net-autostart default
sudo virsh net-start default
```

---

##  3. Create a Virtual Machine (Basic)

Example for Fedora 42:

```bash
virt-install \
  --name testvm \
  --vcpus 8 \
  --memory 8192 \
  --disk path=/var/lib/libvirt/images/testvm.qcow2,size=40 \
  --os-variant fedora-42 \
  --cdrom ~/Downloads/Fedora-Workstation-Live-x86_64-42-1.6.iso \
  --network network=default \
  --graphics spice
```

---

##  4. NUMA-Aware CPU and Memory Configuration

NUMA configuration is not exposed by default in virt-manager, but you can define it manually in the domain XML.

### Edit the VM definition:

```bash
sudo virsh edit testvm
```

### Add NUMA topology inside `<cpu>` and `<memoryBacking>` sections:

```xml
<cpu mode='host-passthrough'>
  <numa>
    <cell id='0' cpus='0-3' memory='4096' unit='MiB'/>
    <cell id='1' cpus='4-7' memory='4096' unit='MiB'/>
  </numa>
</cpu>

<memoryBacking>
  <source type='memfd'/>
  <access mode='shared'/>
  <allocation mode='immediate'/>
</memoryBacking>
```

This setup:

* Creates 2 NUMA nodes
* Assigns 4 vCPUs and 4 GB RAM per node
* Uses `memfd` for shared memory performance

Restart the VM:

```bash
sudo virsh destroy testvm
sudo virsh start testvm
```

Validate:

```bash
sudo virsh vcpuinfo testvm
sudo virsh dommemstat testvm
```

---

##  5. Advanced Customization Examples

###  Pin vCPUs to Physical Cores

```bash
sudo virsh vcpupin testvm 0 0
sudo virsh vcpupin testvm 1 1
sudo virsh vcpupin testvm 2 2
sudo virsh vcpupin testvm 3 3
```

###  Use Hugepages for Better Performance

Add to VM XML:

```xml
<memoryBacking>
  <hugepages>
    <page size='1' unit='GiB'/>
  </hugepages>
</memoryBacking>
```

Enable hugepages on host:

```bash
sudo sysctl -w vm.nr_hugepages=8
```

---

##  6. Manage the VM

```bash
# Start / stop / autostart
sudo virsh start testvm
sudo virsh shutdown testvm
sudo virsh autostart testvm

# Connect to console
sudo virsh console testvm
```

Or use **virt-manager** GUI.

---

##  7. Optional: Bridge Networking

To connect your VM to the external LAN:

### Fedora example

```bash
sudo nmcli connection add type bridge ifname br0 con-name br0
sudo nmcli connection add type ethernet ifname enp1s0 master br0
```

Then modify VM XML:

```xml
<interface type='bridge'>
  <source bridge='br0'/>
  <model type='virtio'/>
</interface>
```

---

##  8. Clone or Template VMs

```bash
sudo virt-clone --original testvm --name devvm --auto-clone
```

---

##  9. Delete VM

```bash
sudo virsh destroy testvm
sudo virsh undefine testvm --remove-all-storage
```

{{< comments >}}
