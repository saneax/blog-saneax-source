---
title: "Learn Datacenter Networking on Your Laptop — Part 3: iPerf3, Fiber, and Juniper — Testing Like a Pro"
description: "Datacenter bandwidth testing with iPerf3, fiber optics troubleshooting, Juniper switch diagnostics, and real throughput testing in GNS3"
slug: "learn-datacenter-networking-gns3-part3"
date: "2026-04-29"
tags: ["iperf3", "networking", "juniper", "fiber", "datacenter", "gns3", "tutorial-series"]
---

**Series: Learn Datacenter Networking on Your Laptop**

- **[Part 1: GNS3 Foundation & Linux Networking Primer](https://blog.saneax.in/tech/learn-datacenter-networking-gns3-part1/)** — Bridges, taps, host networking setup, your first lab
- **[Part 2: VLANs, Bonding, and Network Segregation](https://blog.saneax.in/tech/learn-datacenter-networking-gns3-part2/)** — VLANs, NIC bonding, LACP, bridge isolation
- **Part 3** ← You are here — iPerf3, fiber troubleshooting, Juniper switch diagnostics, throughput testing

---

I spent last weekend debugging a link between two Juniper switches that was running at 2Gbps on a 10G interface. The fiber was plugged in, the lights were blinking, and both switches happily reported "Link is up." Everything looked fine. Except it wasn't.

Turned out — dirty connector on one end. A single microscopic speck of dust. Cleaned it with a one-clicker, link went to 10G. — Two hours. Never getting them back.

I'm writing this so you don't have to go through the same nonsense. This covers everything from basic iPerf3 testing to deep Juniper config, fiber troubleshooting (with DOM readings), and setting up a full lab in GNS3 to practice. Given a choice, I'd love to test this on Arista or Cisco gear too — but our lab runs Juniper, so Juniper it is.

## iPerf3: The Right Way

First things first — get the latest iPerf3. As of early 2026, we're on **v3.21** (released April 2026). If you're running whatever came with your distro's package manager, you're probably on something ancient like v3.1 or v3.9. Ubuntu's repos lag behind terribly.

The big deal with modern iPerf3: **it's finally multi-threaded.** Pre-v3.16, every parallel stream ran on the same CPU core, which meant you'd hit a CPU wall at around 7-8Gbps on most servers no matter what your link could do. If you see a blog post from 2020 saying "iPerf3 is single-threaded," they're not wrong for that era — but it's outdated now.

### The commands you actually need

```bash
# Start server in daemon mode
iperf3 -s -D

# 10G TCP test. Basic, works most of the time
iperf3 -c 10.0.0.1   -t 30 -i 1

# 10G with 4 parallel streams
iperf3 -c 10.0.0.1 -t 30 -i 1 -P 4

# 25G/40G — you need larger buffers and more streams
iperf3 -c 10.0.0.1 -t 30 -i 1 -P 8 -w 2M

# 100G — zerocopy, CPU affinity, proper window
iperf3 -c 10.0.0.1 -t 30 -i 1 -P 16 -Z -A 2,2

iperf3 -c 10.0.0.1 -t 30 -i 1 -P 16 -Z -A 2,2
```

A few things I learned the hard way:

**The `-w` flag sets your TCP window.** A lot of old guides recommend 256K. That was fine for 1G networks. For 10G+, calculate based on your bandwidth-delay product. Rule of thumb: buffer >= bandwidth × RTT. For a 10G link with 1ms latency, you need about 1.25MB. For 40G with 5ms latency, you need about 25MB. If your tests plateau below line rate, bump the window.

**UDP testing with `-u`** tells you different things than TCP. With TCP you're measuring real application throughput including congestion control. With UDP you're measuring the raw capacity of the path — but you have to tell iPerf how much to offer with `-b`. If you say `-b 5G` and the link can actually do 10G, you'll only see 5G. The useful metric here is packet loss and jitter.

```bash
# UDP test — offering 10G to see if the link can handle it
iperf3 -c 10.0.0.1 -u -b 10G -t 30 -i 1
```

**Slow start contaminates short tests.** If you run a 10-second test, half of it is TCP slow start ramping up. Use `-O 3` or `-O 5` to omit the first few seconds. Or just run tests for 30-60 seconds.

**CPU bottleneck is still the #1 problem.** Even with multi-threaded iPerf, at 40G+ you need to pin CPUs. `-A 2,2` pins sender and receiver to core 2. Use `htop` or `mpstat` to check if iPerf is eating a core. If CPU utilization in the JSON output (`-J`) is >80%, you're CPU-bound.

What a good 10G result looks like:

```
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-30.00  sec  35.0 GBytes  10.0 Gbits/sec  0    sender
[  5]   0.00-30.00  sec  35.0 GBytes  10.0 Gbits/sec       receiver
```

Line rate. Zero retransmits. That's what you want.

What's wrong:

- Throughput < 60% of line rate → TCP window too small, CPU bound, or congestion
- Retransmits > 0.1% → actual packet loss on the path
- Wildly fluctuating numbers → QoS shaping or bufferbloat

## Fiber: The Physical Layer Nobody Checks

Look, I get it. Fiber is annoying to troubleshoot because you can't just look at it and tell if it's working. Copper tells you — link down, link up, blink pattern. Fiber just sits there with its little green light and lies to you.

### Know your fiber

Quick reference because I always forget which color is what:

- **OM1** (62.5μm, orange) — legacy trash, avoid for anything modern
- **OM2** (50μm, orange) — also legacy
- **OM3** (50μm, aqua) — laser-optimized, good for 100m at 10G
- **OM4** (50μm, aqua or violet) — datacenter standard, 150m at 100G
- **OM5** (50μm, **lime green**) — SWDM capable, same reach as OM4
- **OS1/OS2** (9μm, yellow) — single-mode, goes for kilometers

Don't mix multi-mode and single-mode. The cores are 50μm vs 9μm — they physically can't couple. I've seen someone jam a yellow patch cable into an aqua port and wonder why it didn't work. The connectors are the same LC form factor so it fits. But it doesn't work.

### The 70% problem

Industry stat: **over 70% of fiber issues are dirty connectors.** Not hardware failure, not bad transceivers, not configuration errors. Dust.

A 1μm particle on a fiber end-face can cause enough back-reflection to bring down a 100G link. You can't see it with your eyes. You need a fiber inspection scope. Every datacenter tech should have one within arm's reach.

Never, ever blow on a connector. Saliva and moisture attract more dust. Use a one-click cleaner or isopropyl alcohol with lint-free wipes.

### Transceiver diagnostics (DOM/DDM)

The single most useful Juniper command for fiber troubleshooting:

```junos
> show interfaces diagnostics optics xe-0/0/0
```

This reads the digital optical monitoring data straight from the SFP+. Here's what to look for:

```
Temperature: 42°C       (0-70°C range, keep under 60°C)
Voltage: 3.30V          (within 5% of spec)
TX Power: -2.5 dBm      (should be in transmit range)
RX Power: -4.2 dBm      (must be above sensitivity threshold)
Laser Bias Current: 6.5 mA  (high alarm threshold means laser is aging)
```

RX Power at or near the sensitivity threshold → clean your connectors. Not replace the transceiver. Clean first. I keep saying this because I've replaced ten transceivers before I learned to clean the fiber.

Common power ranges for reference:

| Transceiver | TX Power | RX Sensitivity | RX Overload |
|---|---|---|---|
| 10G SFP+ SR (MMF, 850nm) | -7.3 to -1.0 dBm | < -11.1 dBm | 0 dBm |
| 25G SFP28 SR (MMF, 850nm) | -6.5 to +2.0 | < -10.0 | +2.0 |
| 40G QSFP+ SR4 (MMF) | -7.3 to +2.0 | < -9.5 | +2.0 |
| 100G QSFP28 LR4 (SMF) | -4.3 to +4.5 | < -11.5 | +4.5 |

If RX Power is above overload — that's too much light. Your receiver is saturated. Add an inline attenuator (3dB or 5dB usually does it).

Also, if your transceiver shows `DOM not supported` in the optics diagnostics output, it's an unsupported 3rd-party optic. Recent JunOS versions have better third-party support, but it's not guaranteed.

### Common fiber issues cheat sheet

| Symptom | Likely Cause | Fix |
|---|---|---|
| Link up but CRC errors | Dirty connector or bend | Inspect + clean, check bend radius |
| RX Power near alarm threshold | Dirty, bad splice, or long distance | Clean first, then check link budget |
| TX Power way off | Bad SFP or DOM reporting error | Replace SFP |
| Link flaps intermittently | Loose connector, bad patch cable | Re-seat, replace patch |
| RX Power = -40 dBm or 0.0 | No signal, broken fiber, unplugged | Check both ends |
| High bias current | Laser aging | Replace transceiver soon |

### Polish mismatch — easy to miss

Three polish types: PC (blue-ish), UPC (blue), APC (green). UPC and APC don't mix. The 8° angle on APC connectors causes high reflection loss if mated with a flat PC/UPC connector. If you force it, you can permanently damage both connectors. APC connectors are green — if you see green, make sure the other end is also green.

## Juniper Switch Configuration: Setting It Up Right

Our lab runs Juniper EX and QFX series. If I could test on Arista or Cisco I would, but you work with what you've got. JunOS is fine once you get used to the `set` / `edit` / `commit` flow.

### Port naming in JunOS

```
xe-0/0/0 → 10G SFP+
et-0/0/0 → 25G, 40G, or 100G (depends on the hardware)
ge-0/0/0 → 1G (usually from a copper SFP or certain optics)
```

### Config templates

**10G SFP+:**

```
set interfaces xe-0/0/0 description "Server-01 - 10G Link"
set interfaces xe-0/0/0 speed 10g
set interfaces xe-0/0/0 link-mode full-duplex
set interfaces xe-0/0/0 mtu 9192
set interfaces xe-0/0/0 unit 0 family ethernet-switching
```

**25G SFP28:**

```
set interfaces et-0/0/0 description "Storage-Node - 25G"
set interfaces et-0/0/0 speed 25g
set interfaces et-0/0/0 link-mode full-duplex
set interfaces et-0/0/0 mtu 9192
set interfaces et-0/0/0 ether-options auto-negotiation
set interfaces et-0/0/0 unit 0 family ethernet-switching
```

**40G/100G:**

```
set interfaces et-0/0/0 description "Spine-01 Uplink"
set interfaces et-0/0/0 speed 100g
set interfaces et-0/0/0 link-mode full-duplex
set interfaces et-0/0/0 mtu 9192
set interfaces et-0/0/0 unit 0 family ethernet-switching
```

**Breakout 40G to 4x10G** (useful when you have a 40G port but your downstream devices only support 10G):

```
set interfaces et-0/0/0 speed 40g
# Then each breakout channel becomes a logical 10G port
# et-0/0/0:0 through et-0/0/0:3
set interfaces et-0/0/0:0 unit 0 family ethernet-switching
set interfaces et-0/0/0:1 unit 0 family ethernet-switching
# ... and so on
```

You need a QSFP+ breakout cable (one QSFP to four LC pairs or four SFP+ ports depending on the cable type).

### The auto-negotiation debate

A lot of old-school network engineers will tell you to disable auto-negotiation on fiber. They're wrong for modern gear.

- IEEE 802.3 Clause 73 defined auto-negotiation for 10G fiber back in the 2000s. Keep it on.
- IEEE 802.3by **requires** auto-negotiation for 25G. Disabling it can actually prevent the link from coming up.
- The advice to disable it came from the 100BASE-FX era where auto-negotiation genuinely didn't work. That's thirty years ago.

Keep auto-negotiation enabled. If you're having link issues, don't start by disabling it. Start by checking the fiber.

### MTU consistency is a trap

JunOS configures layer-2 MTU. If you set `mtu 9192`, that includes the Ethernet header. Your Linux server with `mtu 9000` and your Juniper switch with `mtu 9192` are actually compatible — the Juniper side includes the 14-byte Ethernet header plus 4-byte VLAN tag overhead. But if any device in the path has a different MTU, you'll get dropped packets and ICMP unreachables. Check the whole path.

### Flow control

For regular Ethernet, the default flow control (PAUSE frames) is fine. If you're running FCoE or RoCEv2, you need PFC (Priority Flow Control) instead:

```
set interfaces et-0/0/0 ether-options ieee-802.3-pfc
```

Don't enable PFC unless you actually need it. It can cause head-of-line blocking if misconfigured.

### Show commands for troubleshooting

These are the ones I use most:

```junos
# Basic interface status
> show interfaces xe-0/0/0
> show interfaces xe-0/0/0 terse

# Deep dive — errors, CRC, giants, runts
> show interfaces xe-0/0/0 extensive

# Optics diagnostics (DOM)
> show interfaces diagnostics optics xe-0/0/0

# Error stats
> show interfaces xe-0/0/0 statistics

# See CRC errors specifically
> show interfaces xe-0/0/0 extensive | match "CRC|Error|FCS|alignment"

# LLDP neighbors
> show lldp neighbors
> show lldp neighbors interface xe-0/0/0

# Real-time interface monitoring
> monitor interface traffic
```

The `extensive` output is where you find the real story. If you see CRC errors, FCS errors, or alignment errors, you have a physical layer problem. Don't restart the port — go clean the fiber.

## Lab It Up: GNS3 Setup

Theory is fine but you should never touch a production switch without testing configs first. GNS3 is how you do that without breaking real things.

### Current version and install

As of mid-2026, GNS3 is at **v3.1.0** (the latest tagged release is v2.2.59 with web-ui support, and the v3.1.0 alpha is available for the brave). The v3.x series adds MCP support — basically, you can now hook AI agents into your GNS3 labs for automated testing. Which is cool but optional.

Installation on Ubuntu is straightforward:

```bash
sudo add-apt-repository ppa:gns3/ppa
sudo apt update
sudo apt install gns3-gui gns3-server
```

Or if you prefer Docker:

```bash
docker pull gns3/gns3-server:latest
docker run -d --name gns3-server -p 3080:3080 -v gns3-data:/opt/gns3 gns3/gns3-server:latest
```

Web UI at http://localhost:3080.

For serious labs (multiple virtual routers), use the GNS3 VM. It offloads the heavy lifting to a dedicated VMware or VirtualBox VM. The local server approach works for small topologies but starts choking around 5-6 virtual devices.

### Loading Juniper into GNS3

This used to be complicated. People would mess with Juniper Olive (which is both illegal and based on decade-old code). Don't bother.

Juniper now offers **vJunos** — free for non-commercial use. There's vJunos-switch, vJunos-router, and vJunos-evolved. You need a free Juniper account to download. The images are QEMU/KVM `.qcow2` format.

Alternatively, **vSRX 3.0** is available as a 60-day trial. It's based on JunOS 22.x and runs well in GNS3 with 2-4GB RAM.

Steps:

1. Download the `.qcow2` file from Juniper's download portal
2. Place it in `~/GNS3/images/QEMU/`
3. In GNS3 GUI: Edit → Preferences → QEMU → New
4. Template type: Juniper vSRX / vJunos-switch
5. RAM: 2-4GB (more for complex labs)
6. Console: telnet or VNC

Most guides from 2020 or earlier reference vSRX 2.0. Don't use that — it's end-of-life. The vSRX 3.0 image is what you want.

### Setting up iPerf tests in GNS3

JunOS doesn't include iPerf. You can't SSH into a vJunos switch and run IPerf — the JunOS CLI doesn't have it. What you can do:

1. Add Ubuntu/Docker containers as endpoints in your topology
2. Install iPerf3 in the containers
3. Connect them through your virtual Juniper topology

```bash
# In the GNS3 docker container
iperf3 -s -D   # On the server side
iperf3 -c <server-ip> -t 30 -i 1   # On the client side
```

Your topology looks like:

```
[iperf3-server] --- [vJunos-switch-1] --- (fiber link) --- [vJunos-switch-2] --- [iperf3-client]
```

Right-click on the link between the switches, click Edit, configure filters:
- Latency: 5ms (simulates ~1000km of fiber)
- Bandwidth: 500M (cap the link, though this is a software limiter)
- Optionally add jitter or packet loss

You won't get realistic throughput numbers in GNS3. Expect 100-500Mbps max between vJunos instances — you're running QEMU on a desktop CPU, not an ASIC. What you *can* validate:

- That your config is correct (does the port come up?)
- Routing protocol convergence (OSPF, BGP)
- That ACLs and firewall filters work
- That latency and loss configurations affect traffic as expected

For realistic throughput testing, you need real hardware. GNS3 is for proving your config won't melt the network when you push it to production.

## The Part Where I Tell You to Clean Your Connectors

I started this article with a story about a dirty connector. Let me end with the same advice: before you blame the config, before you blame the switch, before you replace the transceiver — **clean the fiber.**

Keep a fiber inspection scope and one-click cleaner in your kit. A single dirty connector can cause CRC errors, link flaps, or throughput far below line rate. It's the networking equivalent of "did you try turning it off and on again" — except it actually works most of the time.

The commands in this article are current as of early 2026. If you're reading this in 2028, the iPerf version will be different and JunOS config syntax might have changed. But the principles — clean fiber, right transceivers, proper buffer sizes — don't age out.
