---
title: "Connecting Virtual Machines Across Hosts Using VXLAN and Open vSwitch on Linux"
date: 2025-11-02T10:00:00+05:30
description: "Learn how to create VXLAN-based virtual networks using Open vSwitch (OVS) to connect VMs running on different physical hosts, even when each host has only one physical NIC."
summary: "Step-by-step guide to configuring Open vSwitch bridges, VXLAN tunnels, and multi-network virtual machines for cross-host communication on Linux. Includes setup for Fedora and Ubuntu with a single NIC per host."
slug: "vxlan-openvswitch-vm-networking"
draft: false
authors: ["Sanjay Upadhyay"]
tags:
  - Virtualization
  - Open vSwitch
  - VXLAN
  - KVM
  - libvirt
  - Linux Networking
  - Multi-NIC
  - Fedora
  - Ubuntu
categories:
  - Infrastructure
  - Network Virtualization
cover:
  image: "/images/posts/vm-networking-across-physical-hosts-1.png"
  alt: "VXLAN network topology connecting virtual machines across two hosts"
  caption: "Connecting VMs across physical hosts using VXLAN tunnels with Open vSwitch"
  relative: true
keywords:
  - VXLAN with Open vSwitch
  - OVS cross-host networking
  - libvirt multi-network configuration
  - Virtual machine bridge setup
  - Single NIC multiple virtual networks
readingTime: true
showToc: true
tocOpen: true
coverCaption: true
series: ["Virtualization Deep Dive"]
---

##  Blog Outline (Content Overview)

### **Introduction**

* Explain why traditional `virbr0` or NAT networks don’t extend across hosts.
* Introduce **VXLAN (Virtual eXtensible LAN)** as an overlay network solution.
* Show how **Open vSwitch** can encapsulate traffic between VMs across physical servers.


### **Prerequisites**

On each host (Host A & Host B):

```bash
sudo apt install openvswitch-switch -y
# or
sudo dnf install openvswitch -y

sudo systemctl enable --now openvswitch
```

Assume:

| Host  | IP           | Physical NIC |
| ----- | ------------ | ------------ |
| hostA | 192.168.1.10 | enp1s0       |
| hostB | 192.168.1.11 | enp1s0       |


### **Create an OVS Bridge**

On both hosts:

```bash
sudo ovs-vsctl add-br br0
sudo ovs-vsctl add-port br0 enp1s0
sudo ip addr flush dev enp1s0
sudo ip addr add 192.168.1.10/24 dev br0   # adjust per host
sudo ip link set br0 up
```


### **Add VXLAN Tunnel Between Hosts**

```bash
# On hostA
sudo ovs-vsctl add-port br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=192.168.1.11 options:key=100

# On hostB
sudo ovs-vsctl add-port br0 vxlan0 -- set interface vxlan0 type=vxlan options:remote_ip=192.168.1.10 options:key=100
```

This creates a **VXLAN overlay** between hostA and hostB using **VNI 100**, carried over the physical NICs.


### **Attach Virtual Machines to OVS Bridge**

When creating VMs via `virt-install`, attach to the OVS bridge:

```bash
virt-install \
  --name vm1 \
  --vcpus 4 --memory 4096 \
  --disk path=/var/lib/libvirt/images/vm1.qcow2,size=20 \
  --network bridge=br0,model=virtio \
  --cdrom ~/Downloads/ubuntu-24.04.iso
```

Each VM interface connects to `br0`, which is VXLAN-enabled.
Now **VM on hostA can ping VM on hostB** as if they were on the same LAN.


### **Adding Multiple Virtual Networks on the Same Physical NIC**

Yes — absolutely possible.
You can multiplex multiple isolated virtual networks over **one NIC** using:

* Multiple **VXLAN VNIs**, or
* **802.1Q VLAN sub-interfaces**, or
* Multiple OVS bridges each using tagged traffic.

Example:
Create two logical networks (`vxlan100`, `vxlan200`) for two tenant segments:

```bash
sudo ovs-vsctl add-port br0 vxlan100 -- set interface vxlan100 type=vxlan options:key=100 options:remote_ip=192.168.1.11
sudo ovs-vsctl add-port br0 vxlan200 -- set interface vxlan200 type=vxlan options:key=200 options:remote_ip=192.168.1.11
```

Then, define two libvirt networks mapping to those VXLANs:

```bash
<network>
  <name>vxlan100-net</name>
  <forward mode='bridge'/>
  <bridge name='br-vxlan100'/>
</network>
```

Attach different VMs to different VXLANs for isolation.


### **Optional: Add a Management Network (Same NIC)**

If you want separate management and data planes:

```bash
sudo nmcli connection add type vlan ifname enp1s0.10 dev enp1s0 id 10
sudo nmcli connection up enp1s0.10
```

Assign VM management traffic to this VLAN, and keep VXLAN for data.


### **Validate the Setup**

Inside each VM:

```bash
ip addr show
ping <VM on other host>
```

On host:

```bash
sudo ovs-vsctl show
sudo ovs-ofctl dump-flows br0
```

You should see VXLAN encapsulated flows between hosts.


### **Troubleshooting Tips**

* Ensure MTU accommodates VXLAN overhead (`mtu 1450` or lower).
* Check firewall (`UDP/4789` must be open).
* If using NetworkManager, set interface as unmanaged:

  ```bash
  nmcli dev set enp1s0 managed no
  ```


### **What’s Next**

> *“Building a Multi-Host Virtual Data Center Overlay Network using VXLAN and OVS with libvirt automation.”*


{{< comments >}}
