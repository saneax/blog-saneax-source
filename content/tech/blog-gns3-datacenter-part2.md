---
title: "Learn Datacenter Networking on Your Laptop — Part 2: VLANs, Bonding, and Network Segregation"
description: "VLAN tagging, NIC bonding (all 7 modes), LACP with Juniper vJunos-switch in GNS3, and bridge isolation for management, storage, and data networks."
slug: "learn-datacenter-networking-gns3-part2"
date: "2026-04-24"
tags: ["gns3", "networking", "vlan", "bonding", "lacp", "juniper", "datacenter", "bridge", "tutorial-series"]
draft: false
---

**Series: Learn Datacenter Networking on Your Laptop**

- **[Part 1: GNS3 Foundation & Linux Networking Primer](https://blog.saneax.in/tech/learn-datacenter-networking-gns3-part1/)** — Bridges, taps, host networking setup, your first lab
- **Part 2** ← You are here — VLANs, NIC bonding, network segregation
- **[Part 3: iPerf3, Fiber, and Juniper — Testing Like a Pro](https://blog.saneax.in/tech/learn-datacenter-networking-gns3-part3/)** — Throughput testing, fiber diagnostics, Juniper config

---

Here's a thing about real datacenter networks nobody warns you about: **you'll spend more time debugging VLANs and bonding than you will configuring routing.**

VLAN tags that go missing on a trunk. A bonded link where one slave drops packets silently. An interface that's up but the bridge table doesn't have the MAC you need. These are the problems that make you question your life choices at 2 AM.

Part 1 got you comfortable with bridges, taps, and the mental model of GNS3 network simulation. Now we layer on the stuff that makes datacenter networks different from your home setup: VLAN isolation, NIC bonding for redundancy and throughput, and network segregation by traffic type.

## VLANs in Linux — The Right Way

You probably know what a VLAN is at this point. Tagged frames, 802.1Q, 12-bit VLAN ID, 4096 max. Theory is easy. Getting it right in Linux without pulling your hair out — that's the trick.

### Creating VLAN interfaces

In Linux, VLAN subinterfaces look like `eth0.10` or `enp0s3.100`. The dot notation is just convention — you can name them whatever you want with `ip link`. But nobody does that. Stick with the dots.

```bash
# Load 8021q module if not loaded
sudo modprobe 8021q
lsmod | grep 8021q

# Create a VLAN subinterface
ip link add link enp0s3 name enp0s3.10 type vlan id 10

# Add an IP to it and bring it up
ip addr add 10.0.10.1/24 dev enp0s3.10
ip link set enp0s3.10 up

# Verify
ip -d addr show enp0s3.10
```

The `-d` flag matters. Without it, `ip addr show` won't tell you it's a VLAN interface. You'll just see an interface with an IP and wonder why it acts funny. Ask me how many times I've done that.

### VLANs on a bridge

Real datacenter networks don't put IPs on raw interfaces. You bridge everything. Here's where it gets interesting — Linux bridges and VLANs have a subtle relationship.

```bash
# Create a bridge
ip link add name br-mgmt type bridge

# Add a physical interface as a trunk port
ip link set enp0s3 master br-mgmt

# By default, the bridge forwards all VLANs on trunk ports
# To restrict it:
bridge vlan add vid 10 dev enp0s3
bridge vlan add vid 20 dev enp0s3
bridge vlan del vid 1 dev enp0s3  # Remove default VLAN 1

# Check which VLANs are allowed
bridge vlan show
```

This is the part I got wrong for months. I assumed that if you add an interface to a bridge, it automatically carries all VLANs. It does, by default. But if you start adding explicit VLANs with `bridge vlan add`, **the bridge switches to explicit-only mode** — meaning it drops everything except what you've explicitly allowed. That's caught me in production twice.

### GNS3 VLAN setup

In GNS3, VLANs work through the same bridge abstraction. Add a Cloud node, connect it to your topology, and tag the traffic at the switch level inside GNS3.

Topology for this lab:

```
[VPCS-1 (10.0.10.2)]───[vJunos-switch]───[VPCS-2 (10.0.20.2)]
                              │
                         [Cloud node]
                              │
                         [br0 on host]
```

The vJunos-switch acts as the VLAN-aware core. VPCS-1 is in VLAN 10, VPCS-2 in VLAN 20. The Cloud node connects to your host bridge for management access.

On the Juniper side:

```
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members 10
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members 20
set interfaces ge-0/0/2 unit 0 family ethernet-switching vlan members 10
set interfaces ge-0/0/2 unit 0 family ethernet-switching vlan members 20

set vlans VLAN10 vlan-id 10
set vlans VLAN10 interface ge-0/0/0.0
set vlans VLAN10 interface ge-0/0/2.0
set vlans VLAN20 vlan-id 20
set vlans VLAN20 interface ge-0/0/1.0
set vlans VLAN20 interface ge-0/0/2.0
```

ge-0/0/0 and ge-0/0/1 are access ports (single VLAN). ge-0/0/2 is a trunk (both VLANs). VPCS-1 and VPCS-2 won't be able to ping each other directly — they're in different VLANs. That's the whole point.

**Check your work on the Juniper:**

```
show vlans
show ethernet-switching table
show interfaces ge-0/0/2 extensive | match "VLAN"
```

If you see both VLAN 10 and 20 on ge-0/0/2, your trunk is working. If only one shows up, you forgot the second VLAN membership on that interface. Happens to the best of us.

### Why this matters

VLAN segregation is how you keep management traffic off your data plane, and storage traffic off both. In a real datacenter, a misconfigured VLAN can mean your iSCSI storage starts broadcasting across your web servers. That's not a fun conversation with the security team.

## NIC Bonding — The 7 Modes

Linux bonding gives you 7 modes. Most documentation lists them like a spec sheet. I'm going to tell you which ones actually matter and which ones are traps.

### mode=0 (balance-rr) — Round Robin

Packets go out sequentially across slave interfaces: first packet on eth0, second on eth1, third on eth0, and so on.

```bash
ip link add bond0 type bond mode balance-rr miimon 100
ip link set eth0 master bond0
ip link set eth1 master bond0
```

**Where it works:** Simple setups where you don't care about packet ordering. Some NAS devices use this for iSCSI multipathing.

**Where it breaks:** Almost everywhere else. Out-of-order delivery wrecks TCP performance because your receiver keeps asking for retransmits. The throughput looks great on paper and terrible in practice.

**Verdict:** Avoid unless you have a specific reason. I've seen it cause more problems than it solves.

### mode=1 (active-backup) — Hot Standby

One interface active, one waiting. Zero load balancing. Pure redundancy.

```bash
ip link add bond0 type bond mode active-backup miimon 100
ip link set eth0 master bond0
ip link set eth1 master bond0

# Check which slave is active
cat /sys/class/net/bond0/bonding/active_slave
```

**Where it works:** Management interfaces where you care more about uptime than throughput. Server BMCs. Router control plane links.

**Where it breaks:** Anywhere you need more than one link's worth of bandwidth. This is redundancy only, no throughput gain.

**The gotcha:** Failover isn't instant. It depends on your `miimon` interval and the upstream switch's STP convergence. Default 100ms miimon means worst-case 200ms failover. In practice I've seen up to a second.

**Verdict:** Useful. Keep it in your toolbox for management links.

### mode=2 (balance-xor) — XOR Hash

Uses source MAC XOR destination MAC to pick the slave. Same pair of MACs always use the same slave.

```bash
ip link add bond0 type bond mode balance-xor miimon 100 xmit_hash_policy layer2
```

**Where it works:** Simple load balancing without LACP. Good when you have many unique MAC pairs going through the bond.

**Where it breaks:** If you only have two servers talking to each other, they share the same MAC pair — all traffic goes over one slave. Zero load balancing.

**Verdict:** Legacy mode. Use LACP (mode=4) instead unless you can't.

### mode=3 (broadcast) — All Slaves

Every packet goes out on every slave. This is as wasteful as it sounds.

```bash
ip link add bond0 type bond mode broadcast miimon 100
```

**Where it works:** Few places. Some very specific clustering setups. I've never used it in production.

**Verdict:** Skip it. Broadcom's own documentation calls this mode "rarely used" which is vendor-speak for "nobody uses this."

### mode=4 (802.3ad) — LACP ❤️

This is the one you want. Dynamic link aggregation using the Link Aggregation Control Protocol (LACP). The switch and the server negotiate which links are part of the bundle.

```bash
# Load the bonding module with 802.3ad
ip link add bond0 type bond mode 802.3ad miimon 100 lacp_rate fast xmit_hash_policy layer3+4

# Add slaves
ip link set eth0 master bond0
ip link set eth1 master bond0

# Bring the bond up
ip link set bond0 up
ip addr add 10.0.100.1/24 dev bond0
```

**Why it's better:**
- Switch knows which physical ports belong to the bundle
- LACP negotiates: if the switch doesn't support it, the link stays individual (no accidental loops)
- Dynamic failover — the switch learns when a slave goes down
- Load balancing based on your chosen hash policy

**The LACP parameters:**

| Parameter | What it does |
|---|---|
| `miimon 100` | Check link state every 100ms |
| `lacp_rate fast` | Send LACP frames every second (default is slow: 30s) |
| `xmit_hash_policy layer3+4` | Hash on src IP + dst IP + src port + dst port |

**Hash policies for LACP:**

- `layer2` — MAC based. Simple but unbalanced if all traffic goes to one MAC.
- `layer2+3` — MAC + IP. Better distribution. Default on most distros.
- `layer3+4` — IP + port. Best for TCP-heavy workloads. My go-to.

**Switch side (Juniper):**

```
set interfaces ge-0/0/0 gigether-options 802.3ad ae0
set interfaces ge-0/0/1 gigether-options 802.3ad ae0

set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 aggregated-ether-options lacp periodic fast
set interfaces ae0 vlan-tagging
set interfaces ae0 unit 0 family ethernet-switching port-mode trunk
set interfaces ae0 unit 0 family ethernet-switching vlan members 100
set interfaces ae0 unit 0 family ethernet-switching vlan members 200
```

**Verify LACP is talking:**

```
# On Linux
cat /sys/class/net/bond0/bonding/ad_actor_system
cat /proc/net/bonding/bond0 | grep -A 10 "LACP"

# On Juniper
show lacp interfaces
show lacp statistics
show interfaces ae0 extensive
```

**What you want to see:** `LACP state: Active` on both ends. If you see `LACP state: Passive` or no output, check that both sides are configured for LACP active mode.

**Verdict:** Use this for everything that supports it. It's not optional in modern datacenters.

### mode=5 (balance-tlb) — Transmit Load Balancing

Outgoing traffic is balanced based on load. Incoming goes to the current active slave. The kernel rewrites the source MAC on each slave to match the active slave's MAC.

```bash
ip link add bond0 type bond mode balance-tlb miimon 100
```

**Where it works:** Older switches that don't support LACP but you still want some load balancing.

**Where it breaks:** Asymmetric load — transmit is balanced, receive isn't. Switch MAC tables get confused if they see the same MAC on multiple ports.

**Verdict:** Fallback mode. Use LACP if you can.

### mode=6 (balance-alb) — Adaptive Load Balancing

Like balance-tlb, but also balances incoming traffic by ARP negotiation. The bond intercepts ARP replies and rewrites the source MAC to distribute inbound traffic across slaves.

```bash
ip link add bond0 type bond mode balance-alb miimon 100
```

**Where it works:** When you absolutely cannot use LACP and need both Tx and Rx load balancing.

**Where it breaks:** Switches hate it. Some switches will drop packets because they see MAC flapping — same MAC coming from different ports. Also, ARP negotiation means the bond lies to the network about MAC addresses. Elegant hack, but still a hack.

**Verdict:** Impressive trick, but don't put it in production. Use LACP.

### The cheat sheet

| Mode | Load Balancing | Redundancy | Switch Config | Production Ready |
|---|---|---|---|---|
| 0 (balance-rr) | Yes (per packet) | Yes | Static LAG | ❌ TCP hates it |
| 1 (active-backup) | No | Yes | None | ✅ Management |
| 2 (balance-xor) | Yes (per MAC) | Yes | Static LAG | ⚠️ Legacy |
| 3 (broadcast) | No | Yes (redundant) | Static LAG | ❌ Wasteful |
| **4 (802.3ad)** | **Yes (per hash)** | **Yes** | **LACP** | **✅✅✅** |
| 5 (balance-tlb) | Tx only | Yes | None | ⚠️ Fallback |
| 6 (balance-alb) | Tx+Rx | Yes | None | ⚠️ Hackish |

## Bridge Isolation: Keeping Traffic Separate

You don't want your iSCSI storage traffic mixing with your web server traffic. You don't want your backup traffic competing with production. Bridge isolation is how you enforce this.

In GNS3, isolation is straightforward — each "network" in your topology is a separate link between devices, which maps to a separate bridge on the host.

In a real datacenter, it looks like this:

```
┌───────────────────────────────────────────────┐
│                Linux Host                       │
│                                                  │
│  br-mgmt (10.0.1.1/24)    br-data (10.0.10.1/24)│
│      │                          │               │
│   bond0.100                 bond0.200           │
│      │                          │               │
│   └────── bond0 (LACP to switch) ───────┘       │
└───────────────────────────────────────────────┘
```

One physical bond. Three logical networks via VLAN tagging.

```bash
# Create bond interface
ip link add bond0 type bond mode 802.3ad miimon 100

# Create VLAN subinterfaces on the bond
ip link add link bond0 name bond0.100 type vlan id 100
ip link add link bond0 name bond0.200 type vlan id 200
ip link add link bond0 name bond0.300 type vlan id 300

# Create separate bridges for each traffic type
ip link add name br-mgmt type bridge
ip link add name br-data type bridge
ip link add name br-storage type bridge

# Connect VLANs to their respective bridges
ip link set bond0.100 master br-mgmt
ip link set bond0.200 master br-data
ip link set bond0.300 master br-storage

# Assign IPs
ip addr add 10.0.1.1/24 dev br-mgmt
ip addr add 10.0.10.1/24 dev br-data
ip addr add 10.0.20.1/24 dev br-storage

# Bring everything up
ip link set bond0 up
ip link set bond0.100 up
ip link set bond0.200 up
ip link set bond0.300 up
ip link set br-mgmt up
ip link set br-data up
ip link set br-storage up
```

**Check that isolation is working:**

```bash
# From a VPCS in VLAN 100
ping 10.0.1.2  # Should work
ping 10.0.10.1 # Should fail — different VLAN

# Check the bridge tables
bridge fdb show br-mgmt
bridge fdb show br-data
bridge fdb show br-storage
```

If traffic leaks between VLANs, you've either:
- Tagged one of the subinterfaces with the wrong VLAN ID
- The switch doesn't have the correct port VLAN configuration
- STP is temporarily flooding traffic (check with `bridge fdb show` — if you see MACs from VLAN 200 on br-mgmt, something's bridged wrong)

**The GNS3 equivalent:**

In GNS3, you'd build this with three separate Cloud nodes — one for each bridge — or a single Cloud node connected to a Linux bridge that does VLAN filtering. I prefer one Cloud node per bridge because it keeps the topology readable.

## System Tuning for Bonding Performance

Bonding at default settings works. Bonding tuned correctly works better.

### Ring buffer sizing

```bash
# Check current ring buffer
ethtool -g eth0

# Increase (on some NICs)
ethtool -G eth0 rx 4096 tx 4096
```

Default ring buffers are usually 256 or 512 entries. For bonded interfaces under load, 4096 is safer. If you're dropping packets at the NIC level, the ring buffer is the first place to check.

### IRQ affinity

```bash
# Check IRQ for your NIC
cat /proc/interrupts | grep eth0

# Set affinity to specific CPU cores
echo 1 > /proc/irq/$(grep eth0 /proc/interrupts | head -1 | awk -F: '{print $1}')/smp_affinity
```

On multi-core systems, IRQ balancing matters. Default is usually IRQ balance daemon, which works fine. If you're chasing every last megabit, pin NIC IRQs to dedicated cores.

### sysctl tweaks

```bash
# In /etc/sysctl.d/90-bonding.conf

# Increase backlog for bonded interfaces
net.core.netdev_max_backlog = 5000

# Increase TCP buffer limits
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728

# Enable TCP window scaling
net.ipv4.tcp_window_scaling = 1

# Disable slow start after idle (for long-lived bonds)
net.ipv4.tcp_slow_start_after_idle = 0
```

Apply with `sysctl -p /etc/sysctl.d/90-bonding.conf`.

## Debugging Bonding — What to Check First

### Is the bond actually using both links?

```bash
cat /proc/net/bonding/bond0
```

Look for: `Link Failure Count: 0` on each slave. If you see failures, check the physical link.

### Is LACP negotiating?

```bash
# Check LACP state
cat /sys/class/net/bond0/bonding/ad_actor_system
cat /sys/class/net/bond0/bonding/ad_partner_system

# If blank, LACP isn't talking
```

### Is traffic actually balancing?

```bash
# Watch per-slave stats
watch -n 1 'cat /proc/net/bonding/bond0 | grep -A 2 "Slave Interface"'

# Or use ip -s
ip -s link show eth0
ip -s link show eth1
```

One slave doing all the work means your hash policy isn't distributing well. Switch to `layer3+4` hash policy.

### Are there CRC errors?

```bash
ethtool -S eth0 | grep -i crc
```

CRC errors on a bonded link mean physical layer issues — bad cable, bad SFP, or dirty connector. The same dirty fiber problem I ranted about in Part 3 can tank your LACP bond.

## Putting It Together: A Three-VLAN Bonded Lab in GNS3

This is my default lab topology. It's what I use to test configs before touching production gear.

**Topology:**

```
[VPCS-10 (10.0.10.2)]  [VPCS-20 (10.0.20.2)]  [VPCS-30 (10.0.30.2)]
         |                      |                       |
    [vJunos-1]─────────────[vJunos-1]─────────────[vJunos-1]
         |          LACP bond (ge-0/0/0 + ge-0/0/1)
         |
    [Cloud node]
         |
    [bond0 on host (br-mgmt, br-data, br-storage)]
```

**Juniper side for LACP + VLANs:**

```
# LACP bond
set interfaces ge-0/0/0 gigether-options 802.3ad ae0
set interfaces ge-0/0/1 gigether-options 802.3ad ae0

# VLAN tagging on the aggregated interface
set interfaces ae0 vlan-tagging
set interfaces ae0 unit 0 vlan-id 10
set interfaces ae0 unit 0 family ethernet-switching port-mode access
set interfaces ae0 unit 0 vlan-id 20
set interfaces ae0 unit 1 family ethernet-switching port-mode access
set interfaces ae0 unit 1 vlan-id 30
set interfaces ae0 unit 2 family ethernet-switching port-mode access

# Access ports for VPCS
set interfaces ge-0/0/2 unit 0 family ethernet-switching vlan members 10
set interfaces ge-0/0/3 unit 0 family ethernet-switching vlan members 20
set interfaces ge-0/0/4 unit 0 family ethernet-switching vlan members 30

# VLAN definitions
set vlans VLAN10 vlan-id 10
set vlans VLAN20 vlan-id 20
set vlans VLAN30 vlan-id 30
```

**Test it:**

On the host:
```bash
# Ping each VLAN's gateway
ping -c 3 10.0.10.1  # Via br-mgmt
ping -c 3 10.0.20.1  # Via br-data (should work from Juniper)
ping -c 3 10.0.30.1  # Via br-storage

# Isolate check: try cross-VLAN ping from VPCS-10 to VPCS-20
# This should FAIL — they're in different VLANs
```

If VPCS-10 can ping VPCS-20, you've got a bridging loop or VLAN leak. Check your Juniper config. The most common mistake: forgetting the `vlan-tagging` statement on the aggregated Ethernet interface.

One weird thing I've noticed: GNS3 sometimes doesn't forward LACP frames properly between the Cloud node and the Juniper VM. If LACP negotiation fails on the first try, restart both the Juniper and the Cloud node. Don't ask me why that works. It just does.

Quick tangent — I was going to add a section on LACP load balancing algorithms and packet reordering. But honestly, that's a whole other article. The kernel bonding documentation covers it better than I ever could. Check the links below.

Anyway. Back to it.

## What Should Be Automatic By Now

By the end of this part, you should be able to:

1. Create VLAN subinterfaces on Linux and add them to bridges
2. Configure LACP bonding — both on the Linux side and on Juniper
3. Know which bonding mode to use for which situation
4. Set up separate bridges for management, data, and storage traffic
5. Debug a LACP bond: check negotiation, hash distribution, and per-slave errors
6. Build a GNS3 lab with multiple VLANs and a LACP bond to simulate a real datacenter topology

**Next in the series:** [Part 3: iPerf3, Fiber, and Juniper — Testing Like a Pro](https://blog.saneax.in/tech/learn-datacenter-networking-gns3-part3/) picks up from here — now that you've got a functional GNS3 network with VLANs and bonding, go measure its actual throughput, diagnose fiber issues, and tune Juniper switch configs.
