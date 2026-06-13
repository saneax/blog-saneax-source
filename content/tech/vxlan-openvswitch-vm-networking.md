---
title: "VXLAN, OVS, and DPDK in KVM/libvirt: Building a Multi-Host VM Overlay Network"
date: 2026-06-13T22:40:00+05:30
aliases: ["/projects/vxlan-openvswitch-vm-networking/"]
description: "A comprehensive guide to connecting VMs across physical hosts using VXLAN tunnels over Open vSwitch, with KVM/libvirt integration, DPDK-accelerated NICs pinned to NUMA nodes, and huge page configurations."
summary: "Deep dive into building a multi-host overlay network: VXLAN + OVS tunnels between hosts, libvirt network definitions, QEMU command-line DPDK setup with vhost-user, NUMA-aware vCPU pinning, and practical validation techniques. Designed for home lab experimentation."
slug: "vxlan-openvswitch-vm-networking"
draft: false
authors: ["Sanjay Upadhyay"]
tags:
  - "tech"
  - "dpdk"
  - "vxlan"
  - "openvswitch"
  - "kvm"
  - "libvirt"
  - "virsh"
  - "numa"
  - "vhost-user"
  - "hugepages"
  - "linux-networking"
  - "home-lab"
  - "aarch64"
  - "virtualization"
categories:
  - Engineering
  - Infrastructure
  - Network Virtualization
cover:
  image: ""
  alt: "VXLAN + OVS + DPDK overlay network topology"
  caption: "Multi-host virtual network fabric using VXLAN tunnels, OVS bridging, and DPDK-accelerated guest NICs"
  relative: true
keywords:
  - VXLAN Open vSwitch KVM
  - DPDK vhost-user libvirt
  - NUMA pinning virtual machines
  - OVS-DPDK data path
  - cross-host VM networking
showToc: true
tocOpen: true
series: ["Virtualization Deep Dive"]
author: "Sanjay Upadhyay"
---

The original version of this article covered basic VXLAN tunnels over Open vSwitch for cross-host VM connectivity. That works fine for most use cases, but it leaves the guest network path entirely in kernel space — bridged through the host OVS bridge, traversing kernel TCP/IP stack overhead, and sharing CPU cycles with everything else.

This expanded version goes deeper. We'll cover:

1. The **host-side overlay network** using OVS VXLAN tunnels (from the original guide)
2. **libvirt/virsh integration** — defining persistent virtual networks backed by OVS bridges
3. **DPDK-accelerated guest NICs** — passing DPDK-enabled virtio devices into the VM using `vhost-user`
4. **NUMA-aware vCPU and memory pinning** — isolating the VM onto specific NUMA nodes for deterministic performance
5. **Huge page configuration** — 1G and 2M page backing for guest memory

Everything here is **experimental** — this is a home lab setup designed to test DPDK datapath acceleration with VXLAN overlay networking. Some of these configurations will only apply on suitable hardware (x86_64 with VFIO support, or aarch64 with proper SMMU/IOMMU). The principles, however, are universal.

---

## Part 1: The Underlay — VXLAN Tunnels Over OVS

This is where we start. The physical hosts need an overlay network carrying tenant traffic across the wire. VXLAN (RFC 7348) encapsulates Layer 2 frames in UDP packets over the existing Layer 3 network. Each tenant network gets a 24-bit VNI (VXLAN Network Identifier).

### Prerequisites

On each host:

```bash
# Debian/Ubuntu
sudo apt install openvswitch-switch -y
sudo systemctl enable --now openvswitch-switch

# Fedora/RHEL
sudo dnf install openvswitch -y
sudo systemctl enable --now openvswitch
```

Assume a simple two-host topology:

| Host  | IP Address     | Physical NIC | VM Storage Path       |
|-------|----------------|--------------|-----------------------|
| hostA | 192.168.1.10   | enp1s0       | /var/lib/libvirt/images |
| hostB | 192.168.1.11   | enp1s0       | /var/lib/libvirt/images |

### Step 1: Create the OVS Bridge

On **both** hosts, move the physical NIC into an OVS bridge:

```bash
sudo ovs-vsctl add-br br-ovs
sudo ovs-vsctl add-port br-ovs enp1s0

# Move IP from physical NIC to bridge
sudo ip addr flush dev enp1s0
sudo ip addr add 192.168.1.10/24 dev br-ovs   # adjust per host
sudo ip link set br-ovs up
```

At this point, `br-ovs` has replaced `enp1s0` as the host's primary network interface. Traffic between hostA and hostB flows through the OVS bridge.

### Step 2: Add VXLAN Tunnel Ports

```bash
# On hostA — point to hostB
sudo ovs-vsctl add-port br-ovs vxlan0 -- \
  set interface vxlan0 type=vxlan \
  options:remote_ip=192.168.1.11 \
  options:key=100

# On hostB — point to hostA
sudo ovs-vsctl add-port br-ovs vxlan0 -- \
  set interface vxlan0 type=vxlan \
  options:remote_ip=192.168.1.10 \
  options:key=100
```

`key=100` is the VNI. Any traffic entering `br-ovs` destined for the remote VXLAN endpoint gets encapsulated with VNI 100.

### Step 3: Verify the Overlay

```bash
sudo ovs-vsctl show
sudo ovs-ofctl dump-flows br-ovs
```

You should see the VXLAN tunnel port and flow rules for both physical and tunnel interfaces. If your firewall is blocking UDP/4789, `tcpdump -i any udp port 4789` on one host while pinging from the other should reveal empty captures — a telltale sign of a firewall rule interfering.

---

## Part 2: libvirt Network Definitions for OVS Bridges

The manual OVS bridge setup is fine for testing. For persistent VM deployments, you want libvirt to manage the network connection. We can define a **virtual network** in libvirt that points to the OVS bridge.

### Define a Persistent OVS Network in libvirt

Create `/etc/libvirt/qemu/networks/ovs-vxlan100.xml` on **each host**:

```xml
<network>
  <name>ovs-vxlan100</name>
  <forward mode='bridge'/>
  <bridge name='br-ovs'/>
  <virtualport type='openvswitch'/>
  <portgroup name='vxlan100' default='yes'>
    <vlan>
      <tag id='100'/>
    </vlan>
  </portgroup>
</network>
```

Activate it:

```bash
sudo virsh net-define /etc/libvirt/qemu/networks/ovs-vxlan100.xml
sudo virsh net-start ovs-vxlan100
sudo virsh net-autostart ovs-vxlan100
```

Now you can attach VMs to this network using:

```bash
virsh attach-interface vm1 --type network --source ovs-vxlan100 --model virtio --config --live
```

Or include it in the VM's domain XML from the start:

```xml
<interface type='network'>
  <mac address='52:54:00:ab:cd:01'/>
  <source network='ovs-vxlan100' portgroup='vxlan100'/>
  <model type='virtio'/>
  <virtualport type='openvswitch'/>
</interface>
```

### Why This Matters

Using libvirt network definitions means:
- The OVS connection is properly tracked by libvirt
- VM migration preserves network attachment
- You can manage VXLAN tags per VM or per portgroup
- Integration with `virt-manager` and other management tools works out of the box

---

## Part 3: DPDK-Accelerated Guest NICs via vhost-user

Now we move into experimental territory. Instead of the standard virtio NIC (which goes through the kernel's vhost_net), we can attach a **vhost-user** interface — a shared-memory ring buffer that lets the guest's virtio driver communicate directly with a userspace DPDK poll-mode driver (PMD) in the host.

The data path comparison:

```
Standard virtio:
  QEMU → tap → kernel bridge/OVS kernel → physical NIC

DPDK vhost-user:
  QEMU vhost-user → OVS-DPDK PMD → physical NIC (userspace all the way)
```

The vhost-user path eliminates kernel context switches and copies for every packet. This is critical for workloads that care about latency (trading, AI inference serving, real-time signal processing).

### Host Requirements

DPDK on the host requires:

1. **IOMMU/VFIO support** — for direct device assignment (or vfio-noiommu for testing)
2. **Huge Pages** — DPDK PMDs allocate packet buffers from huge pages
3. **DPDK-compatible NIC** — Intel 82599/i40e/ice, Mellanox ConnectX, or a kernel driver that supports `uio_pci_generic` / `vfio-pci`

#### Check IOMMU Support

```bash
# x86_64
dmesg | grep -i iommu
cat /proc/cmdline | grep -o 'intel_iommu=on\|amd_iommu=on'

# aarch64
dmesg | grep -i smmu
cat /proc/cmdline | grep -o 'iommu.passthrough=on'
```

If there's no IOMMU (common on Raspberry Pi and lower-end hardware), you can use `vfio-noiommu` for experimentation:

```bash
echo "vfio" >> /etc/modules
echo "vfio-noiommu" >> /etc/modules  # only for test environments
```

#### Allocate Huge Pages

```bash
# Temporary
echo 1024 | sudo tee /proc/sys/vm/nr_hugepages

# Persistent (add to /etc/default/grub)
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=2M hugepagesz=2M hugepages=1024 iommu.passthrough=on"
# Or for 1G pages (x86_64 only):
GRUB_CMDLINE_LINUX_DEFAULT="default_hugepagesz=1G hugepagesz=1G hugepages=4"

sudo update-grub && sudo reboot
```

Verify allocation:

```bash
cat /proc/meminfo | grep -i huge
# HugePages_Total: 1024
# Hugepagesize: 2048 kB
```

Mount hugetlbfs (libvirt/QEMU needs this for vhost-user memory backing):

```bash
sudo mount -t hugetlbfs hugetlbfs /dev/hugepages
# Make persistent:
echo "hugetlbfs /dev/hugepages hugetlbfs defaults 0 0" | sudo tee -a /etc/fstab
```

### OVS-DPDK Integration

To use vhost-user interfaces with OVS, the OVS data path must be configured for DPDK:

```bash
# Set OVS to use DPDK
sudo ovs-vsctl set Open_vSwitch . other_config:dpdk-init=true

# Set aside some core(s) for DPDK PMD threads
sudo ovs-vsctl set Open_vSwitch . other_config:pmd-cpu-mask=0x30  # cores 4,5

# Allocate huge pages for OVS-DPDK
sudo ovs-vsctl set Open_vSwitch . other_config:dpdk-socket-mem="1024"

# Restart OVS
sudo systemctl restart openvswitch-switch
```

Now add a vhost-user port to the bridge:

```bash
sudo ovs-vsctl add-port br-ovs vhost-user-0 -- \
  set Interface vhost-user-0 type=dpdkvhostuserclient \
  options:vhost-server-path=/tmp/vhost.sock
```

This creates a Unix domain socket at `/tmp/vhost.sock`. QEMU connects to this socket as the vhost-user client, and all guest virtio traffic goes through the DPDK PMD in userspace.

### QEMU Command-Line for vhost-user

For a VM with two NICs — one standard virtio for management, one DPDK vhost-user for data plane:

```bash
qemu-system-x86_64 \
  -name vm-dpdk-test \
  -machine accel=kvm,usb=off \
  -cpu host,migratable=off \
  -m 4096 \
  -object memory-backend-file,id=mem,size=4096M,mem-path=/dev/hugepages,share=on \
  -numa node,memdev=mem \
  -smp 4 \
  \
  -drive file=/var/lib/libvirt/images/vm-dpdk.qcow2,if=virtio \
  \
  # Management NIC (standard virtio, goes through kernel)
  -netdev tap,id=mgmt0,ifname=tap-mgmt,script=no,downscript=no,vhost=on \
  -device virtio-net-pci,netdev=mgmt0,mac=52:54:00:aa:bb:01 \
  \
  # Dataplane NIC (vhost-user, goes through DPDK)
  -chardev socket,id=char1,path=/tmp/vhost.sock,server=off \
  -netdev vhost-user,id=dpdk0,chardev=char1,vhostforce=on \
  -device virtio-net-pci,netdev=dpdk0,mac=52:54:00:aa:bb:02,mq=on,vectors=6 \
  \
  -display none -nographic
```

The critical flags:
- `-object memory-backend-file,...share=on` — huge page backing with sharing enabled (vhost-user requires shared memory)
- `-chardev socket,path=/tmp/vhost.sock` — the Unix socket connecting to OVS's vhost-user server
- `mq=on,vectors=6` — multi-queue virtio for the dataplane NIC (6 vectors = 2 queues + config + control)

### libvirt Domain XML for vhost-user

Here's the same configuration in libvirt XML — this is how you'd define it persistently:

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>vm-dpdk</name>
  <memory unit='GiB'>4</memory>
  <currentMemory unit='GiB'>4</currentMemory>
  <memoryBacking>
    <hugepages/>
    <access mode='shared'/>
  </memoryBacking>
  <vcpu placement='static'>4</vcpu>

  <os>
    <type arch='x86_64' machine='pc-q35-9.0'>hvm</type>
  </os>

  <features><acpi/><apic/></features>
  <cpu mode='host-passthrough' migratable='off'/>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <devices>
    <!-- Storage -->
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='/var/lib/libvirt/images/vm-dpdk.qcow2'/>
      <target dev='vda' bus='virtio'/>
    </disk>

    <!-- Management NIC (standard vhost-net) -->
    <interface type='bridge'>
      <mac address='52:54:00:aa:bb:01'/>
      <source bridge='br-ovs'/>
      <model type='virtio'/>
      <driver name='vhost'/>
      <virtualport type='openvswitch'/>
    </interface>

    <!-- Dataplane NIC (vhost-user, DPDK accelerated) -->
    <interface type='vhostuser'>
      <mac address='52:54:00:aa:bb:02'/>
      <source type='unix' path='/tmp/vhost.sock' mode='client'/>
      <model type='virtio'/>
      <driver name='vhost' queues='2'>
        <host mq='on'/>
        <guest mq='on'/>
      </driver>
    </interface>
  </devices>
</domain>
```

Start the VM:

```bash
sudo virsh define vm-dpdk.xml
sudo virsh start vm-dpdk
```

---

## Part 4: NUMA-Aware VM Pinning

When you have a host with multiple NUMA nodes, the worst thing you can do is let QEMU freely schedule vCPUs across nodes. Cross-NUMA memory access adds 30-50% latency. For DPDK dataplane VMs, you must pin vCPUs and memory to the **same NUMA node**.

### Identify Host NUMA Topology

```bash
# Check NUMA nodes and their CPU ranges
lscpu | grep -E "NUMA|Core|Socket|Thread"

# Check memory per NUMA node
numactl --hardware

# On aarch64 (Raspberry Pi 5)
cat /sys/devices/system/node/online  # 0-7
cat /sys/devices/system/node/node*/cpulist  # 0-3 (same 4 CPUs listed for each of 8 nodes)
```

### Pin vCPUs and Memory in libvirt

Modify the domain XML to pin vCPUs to specific physical CPUs and restrict memory to a single NUMA node:

```xml
<domain type='kvm'>
  <name>vm-dpdk-numa</name>
  <memory unit='GiB'>4</memory>
  <currentMemory unit='GiB'>4</currentMemory>
  <memoryBacking>
    <hugepages>
      <page size='2048' unit='KiB' nodeset='0'/>
    </hugepages>
    <access mode='shared'/>
  </memoryBacking>
  <vcpu placement='static'>2</vcpu>
  <iothreads>1</iothreads>

  <cpu mode='host-passthrough'>
    <numa>
      <cell id='0' cpus='0-1' memory='4' unit='GiB'/>
    </numa>
  </cpu>

  <numatune>
    <memory mode='strict' nodeset='0'/>
    <memnode cellid='0' mode='strict' nodeset='0'/>
  </numatune>

  <cputune>
    <vcpupin vcpu='0' cpuset='0'/>
    <vcpupin vcpu='1' cpuset='1'/>
    <emulatorpin cpuset='0'/>
    <iothreadpin iothread='1' cpuset='1'/>
  </cputune>

  <!-- rest of the domain config... -->
</domain>
```

Key elements:
- **`<numatune><memory mode='strict' nodeset='0'/>`** — forces all guest memory to NUMA node 0
- **`<vcpupin>`** — pins each vCPU to a specific physical CPU on the target NUMA node
- **`<emulatorpin>`** — pins the QEMU emulator thread (I/O emulation) to the same NUMA node
- **`<iothreadpin>`** — pins IOThread to NUMA node 0 for disk I/O isolation

### On a Host Without NUMA (or Single-Node ARM)

If your host doesn't have distinct NUMA nodes (most single-socket systems), `numatune` does nothing harmful — it just confirms everything is in node 0. The pinning still improves cache locality. On the Raspberry Pi 5 pictured above, all 4 CPUs are in all 8 NUMA nodes (a flat memory topology), so strict NUMA pinning has no memory latency benefit, but **vCPU pinning** still prevents the host scheduler from bouncing vCPUs between cores.

---

## Part 5: Huge Page Configuration Deep-Dive

Huge pages reduce TLB pressure. A VM with 4GB of RAM backed by 4K pages requires 1,048,576 TLB entries. With 2M huge pages, that drops to 2,048 entries. With 1G pages (x86_64 only), just 4 entries.

### Configure Huge Pages for QEMU/VMs

```bash
# 1G huge pages (x86_64 preferred)
sudo mount -t hugetlbfs hugetlbfs /dev/hugepages1G -o pagesize=1G

# 2M huge pages (works on all architectures)
sudo mount -t hugetlbfs hugetlbfs /dev/hugepages -o pagesize=2M
```

In the libvirt domain XML, the `<memoryBacking>` section handles this:

```xml
<memoryBacking>
  <hugepages>
    <page size='1048576' unit='KiB' nodeset='0'/>  <!-- 1G pages for node 0 -->
  </hugepages>
  <access mode='shared'/>                             <!-- required for vhost-user -->
</memoryBacking>
```

**Common pitfalls:**
- If `<access mode='shared'/>` is missing, vhost-user connections fail silently
- If huge pages are not pre-allocated, memory backing falls back to anonymous pages
- 1G pages require kernel config `CONFIG_HUGETLB_PAGE` and `CONFIG_HUGETLBFS` (both are enabled in most distro kernels)

---

## Part 6: End-to-End Network Flow — What Happens Per Packet

To truly understand this stack, trace one packet path:

```
Guest VM userspace app
    ↓
Guest virtio-net driver → writes descriptors to vring
    ↓
vhost-user client socket (/tmp/vhost.sock)
    ↓
OVS DPDK PMD (userspace) reads from vring via shared memory
    ↓
OVS flow table lookup → matches VXLAN output to br-ovs
    ↓
VXLAN encapsulation (VNI 100, UDP dst port 4789)
    ↓
OVS enqueues packet to physical NIC PMD
    ↓
NIC hardware transmits onto wire
    ↓
[network]
    ↓
Remote host NIC receives → OVS DPDK PMD
    ↓
VXLAN decap → OVS flow lookup → vhost-user socket
    ↓
Remote guest vring → remote guest virtio-net → app
```

**Zero kernel crossings** on the data path. The entire chain stays in userspace once the packet leaves the guest virtio ring. This is why DPDK + vhost-user matters — it's not about making one operation faster; it's about eliminating the kernel as a bottleneck for every operation.

---

## Part 7: Validation and Benchmarking

### Check Connectivity

Inside the VM:

```bash
ip addr show
# The management NIC should have an IP. The DPDK NIC may not if running raw DPDK.
# If running standard networking over the vhost-user NIC:
ip addr add 10.99.0.1/24 dev enp2s0
ip link set enp2s0 up
```

### Test VXLAN Forwarding

From hostA to hostB through the overlay:

```bash
# On the host — check VXLAN encap works
sudo tcpdump -i enp1s0 udp port 4789
```

### Benchmark with DPDK testpmd

Inside a DPDK-aware VM:

```bash
# If guest has DPDK testpmd
dpdk-testpmd -l 0-1 -n 4 -- --txd=512 --rxd=512 \
  --forward-mode=io \
  --stats-period=5
```

Look for:
- **Zero packet loss** — if OVS-DPDK and vhost-user are configured correctly
- **Latency under 50µs** at line rate — achievable with proper NUMA pinning

### Benchmark with iperf3

If using standard networking over both paths:

```bash
# On VM-A
iperf3 -s

# On VM-B
iperf3 -c <VM-A-IP>
```

Compare standard virtio vs vhost-user throughput. The difference is typically minimal at low rates, but under heavy load (multiple flows, many packets per second), vhost-user pulls ahead by 15-30% due to reduced CPU overhead.

---

## Part 8: Troubleshooting the Experimental Stack

This is a complex, multi-layered setup. When something breaks — and it will — here's where to look:

### OVS-DPDK Not Starting

```bash
# Check OVS logs
sudo journalctl -u openvswitch-switch --since "5 minutes ago" | tail -50
sudo ovs-appctl dpdk/log-list | grep -i error

# Verify DPDK kernel modules are loaded
lsmod | grep -E "vfio|uio|igb_uio"

# Check that huge pages are available
cat /proc/meminfo | grep HugePages_Free
```

### vhost-user Connection Failing

```bash
# The socket must exist and be writable by the QEMU process
ls -la /tmp/vhost.sock

# Check OVS shows the vhost-user port
sudo ovs-vsctl list interface vhost-user-0 | grep -i status
# Should show "status: {connected=true}"

# If the QEMU process can't connect, check AppArmor/SELinux
sudo aa-status | grep qemu
sudo ausearch -m avc -ts recent
```

### VXLAN Tunnel Not Forwarding

```bash
# Verify the tunnel port exists and is up
sudo ovs-vsctl list interface vxlan0 | grep -E "type|options|admin_state"

# Check flows
sudo ovs-ofctl dump-flows br-ovs | grep vxlan

# Test connectivity at the underlay level
ping 192.168.1.11  # must work before VXLAN will

# Check MTU — VXLAN adds 50 bytes overhead, reduce MTU on bridges
sudo ip link set br-ovs mtu 1450
```

### NUMA Pinning Not Working

```bash
# Verify CPU pinning is applied
sudo virsh vcpupin vm-dpdk-numa

# Check that the VM's memory is allocated on the right node
sudo virsh numatune vm-dpdk-numa

# Validate with numactl inside the guest (if accessible)
# If the guest has its own NUMA topology, verify alignment
```

---

## What This Setup Is (and Is Not)

**This is:**
- An experimental lab configuration for learning DPDK datapath with VXLAN overlay networking
- A template for testing OVS-DPDK + vhost-user with KVM/libvirt
- A foundation for building more complex multi-host virtual network topologies

**This is not:**
- Production-ready. The vhost-user socket path (`/tmp/vhost.sock`) is fragile. Security considerations (SELinux, AppArmor, socket permissions) are significant.
- A substitute for proper SDN controllers. OVS alone doesn't do automated tunnel endpoint discovery — that's what controllers like OVN, Tungsten Fabric, or OpenDaylight provide.
- Performance-tuned. Every environment's optimal huge page count, PMD core mask, and RX/TX queue depth will differ.

---

## Where to Go From Here

Once you have the basic VXLAN tunnel + OVS bridge + vhost-user VM working, the natural next steps are:

1. **Add a second VXLAN VNI** — create an isolated tenant network (VNI 200) with its own set of VMs
2. **Introduce OVN** — OVN adds distributed logical switches, routers, and ACLs on top of OVS
3. **Try a DPU/NIC-Offloaded setup** — Mellanox ConnectX SmartNICs can offload VXLAN encapsulation to hardware, achieving line rate even at 100G
4. **Integrate with Kubernetes** — OVS + OVN is the network backend for many OpenShift/SDN deployments
5. **Multi-queue RSS** — enable Receive Side Scaling across multiple vhost-user queues for multi-core VMs

The line between "home lab tinkering" and "what hyperscalers do at $BUDGET" is thinner than most people think. AWS Nitro works on similar principles. Google's Andromeda virtual network is built on an OVS-like datapath. The concepts scale — only the hardware budget changes.

{{< comments >}}
