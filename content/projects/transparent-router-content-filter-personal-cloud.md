---
title: "Pi03: Transparent Router, Content Filter & Personal Cloud"
date: 2025-11-23T19:00:00+05:30
draft: false
author: "Sanjayu"
tags: ["Raspberry Pi", "Networking", "Home Lab", "Nextcloud", "AdGuard Home", "Docker", "Linux"]
categories: ["Projects", "Documentation"]
description: "Comprehensive documentation for setting up a Raspberry Pi 4 as a multi-WAN router, transparent content filter, and RAID 1 cloud storage server."
weight: 10
---

# Project Pi03: Transparent Router, Content Filter & Personal Cloud

**Host:** Raspberry Pi 4 Model B (4GB RAM)
**Hostname:** `pi03`
**OS:** Debian 12 (Bookworm)
**Functions:** Multi-WAN Router, AdGuard Home DNS Filter, Nextcloud Storage (RAID 1), Cloudflare Tunnel.
**Date:** November 23, 2025

## 1\. System Architecture & Network Design

This system operates as a transparent router for a LAN subnet while hosting secure personal cloud services accessible from the internet.

### Network Interfaces & Zones

| Interface | Name (Alias) | Type | Role | Subnet/IP | Metric |
| --- | --- | --- | --- | --- | --- |
| **eth0** | `eth0-mgmt` | Built-in | Management (SSH) | DHCP (No Default GW) | 102 |
| **eth1** | **wan0** | USB NIC | **Primary WAN** (Static) | `10.10.72.195/26` | 100 |
| **eth2** | **lan0** | USB NIC | **LAN Gateway** | `192.168.10.1/24` | N/A |
| **wlan0** | **wan1** | Built-in Wi-Fi | **Backup WAN** (DHCP) | DHCP (Metric 200) | 200 |
| **docker0** | `docker0` | Bridge | Docker Default | `172.17.0.1/16` | N/A |
| **br-xx** | `br-xxxx` | Bridge | Compose Network | `172.x.0.1/16` | N/A |

### Traffic Flow

1.  **LAN Clients** (`192.168.10.x`) → **Pi03 Gateway** (`192.168.10.1`).
2.  **DNS Redirection:** All UDP/TCP 53 traffic is transparently redirected via `nftables` to **AdGuard Home** (`192.168.10.1:53`).
3.  **AdGuard Home:** Filters content → Resolves via Upstream (8.8.8.8) using WAN failover.
4.  **Internet Access:** Traffic is NAT'd (Masqueraded) out via `wan0` (Primary) or `wan1` (Backup).

- - -

## 2\. Network Configuration (NetworkManager & Udev)

### A. Persistent Naming Rules (Udev)

_File:_ `/etc/udev/rules.d/70-persistent-net.rules`

```
# Rule 1: Static WAN Interface (Physical Path)
SUBSYSTEM=="net", ACTION=="add", ATTRS{devpath}=="1.1", NAME="wan0"

# Rule 2: LAN Interface (Physical Path)
SUBSYSTEM=="net", ACTION=="add", ATTRS{devpath}=="2.4", NAME="lan0"

# Rule 3: Backup WAN (Internal Wi-Fi MAC Address)
SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="dc:a6:32:4c:72:4b", NAME="wan1"
```

### B. Connection Profiles (nmcli)

#### 1\. Primary WAN (Static)

```
nmcli connection add type ethernet con-name wan0-wan-static ifname wan0 \
  ipv4.method manual ipv4.addresses 10.10.72.195/26 ipv4.gateway 10.10.72.193 \
  ipv4.dns "8.8.8.8 4.2.2.1" ipv4.route-metric 100
```

#### 2\. Backup WAN (Wi-Fi DHCP)

```
# Note: Requires `iw reg set <COUNTRY>` to see 5GHz networks.
nmcli connection add type wifi con-name wan1-wifi-dhcp ifname wan1 \
  ssid "<SSID>" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "<PASSWORD>" \
  ipv4.method auto ipv4.route-metric 200 connection.interface-name wan1
```

#### 3\. LAN Gateway (Static)

```
nmcli connection add type ethernet con-name lan0-lan ifname lan0 \
  ipv4.method manual ipv4.addresses 192.168.10.1/24 ipv4.never-default yes
```

- - -

## 3\. Firewall & Routing (Nftables)

_File:_ `/etc/nftables.conf`

Handles NAT, transparent DNS redirection, and routing policies. **Crucially**, it must load _after_ Docker networks exist.

```
#!/usr/sbin/nft -f
flush ruleset

table ip filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        ip protocol icmp accept
        iif lo accept
        
        # Management Access
        iif eth0 tcp dport 22 accept
        
        # LAN Services (DNS, DHCP, Web UI)
        iif lan0 udp dport { 53, 67 } accept
        iif lan0 tcp dport { 53, 8080, 8081 } accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
        ct state established,related accept
        
        # Allow LAN to WAN (New Connections)
        iif lan0 oif wan0 ct state new accept
        iif lan0 oif wan1 ct state new accept
        
        # Allow Docker to WAN
        iif docker0 oif wan0 accept
        iif docker0 oif wan1 accept
        iif br-f9536f6adf36 oif wan0 accept
        iif br-f9536f6adf36 oif wan1 accept
        
        # Allow Inter-VLAN / LAN-to-LAN
        iif lan0 oif lan0 accept
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        oif wan0 masquerade
        oif wan1 masquerade
    }
    
    chain prerouting {
        type nat hook prerouting priority -100; policy accept;
        # Transparent DNS Redirect
        iif lan0 udp dport 53 redirect
        iif lan0 tcp dport 53 redirect
    }
}
```

### Systemd Override for Nftables

_File:_ `/etc/systemd/system/nftables.service.d/override.conf`

```
[Unit]
After=network-online.target sysctl.service docker.service
Wants=network-online.target sysctl.service
Requires=docker.service
```

- - -

## 4\. Services Configuration

### A. Dnsmasq (DHCP Server)

_File:_ `/etc/dnsmasq.conf`

```
interface=lan0
dhcp-range=192.168.10.100,192.168.10.250,255.255.255.0,12h
dhcp-option=3,192.168.10.1   # Gateway
dhcp-option=6,192.168.10.1   # DNS Server
port=0                       # Disable built-in DNS (AdGuard handles this)
```

### B. AdGuard Home (DNS Filter)

*   **Listening Interface:** `0.0.0.0:53` (Binds all IPs)
*   **Upstream DNS:** `8.8.8.8`, `1.1.1.1` (IPv4 Only for stability)
*   **Important Setting:** `disable_ipv6: true` (Prevents timeout issues in multi-WAN).

### C. Storage (RAID 1)

*   **Device:** `/dev/md0` (Mirrored)
*   **Mount Point:** `/mnt/cloudstorage`
*   **Fstab Entry:** `UUID=<UUID> /mnt/cloudstorage ext4 defaults,nofail 0 2`
*   **Monitoring:** Email alerts configured in `/etc/mdadm/mdadm.conf`.

- - -

## 5\. Nextcloud & Cloudflare Tunnel (Docker)

Hosted in `~/nextcloud-docker/` using `docker-compose`.

### Docker Compose Configuration

*   **App:** Nextcloud (Port 8081).
*   **DB:** MariaDB 10.6 (Uses custom `init.sql` to bypass credential bugs).
*   **Volumes:** Data mounted from `/mnt/cloudstorage/nextcloud_data`.
*   **Cron:** Host-based cron job running `docker exec`.

### Custom Database Init (`init.sql`)

```
CREATE USER 'nextclouduser'@'%' IDENTIFIED BY 'MySecurePassword';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';
FLUSH PRIVILEGES;
```

### Cloudflare Tunnel

*   **Service:** `cloudflared`
*   **Ingress:** Routes `https://cloud.sanstore.saneax.in` → `http://localhost:8081`.
*   **Access:** Secured via HTTPS (Full Mode) at Cloudflare Edge.

- - -

## 6\. Troubleshooting & Recovery Commands

### Network & Routing

*   **Check IP/Interfaces:** `ip a l`
*   **Check Routes (Metric verification):** `ip r l`
*   **Restart Firewall:** `sudo systemctl restart nftables`
*   **Reload Udev Rules:** `sudo udevadm control --reload-rules && sudo udevadm trigger`

### Docker & Services

*   **Check Container Health:** `docker ps`
*   **Check Docker Logs:** `docker logs nextcloud-docker_app_1`
*   **Restart Nextcloud Stack:** `cd ~/nextcloud-docker && sudo docker-compose restart`
*   **Test Internal DNS:** `dig @127.0.0.1 google.com`
*   **Test External DNS:** `dig @8.8.8.8 google.com`

### RAID 1 Maintenance

*   **Check Health:** `cat /proc/mdstat`
*   **Detailed Status:** `sudo mdadm --detail /dev/md0`
*   **Test Email Alert:** `sudo mdadm --monitor --scan --test`

- - -

### 7\. Known Issues & Fixes

*   **"Connection Timed Out" on Nextcloud:**  
    _Cause:_ `nftables` blocking internal Docker bridge traffic.  
    _Fix:_ Ensure `forward` chain has `iif br-xxxx oif br-xxxx accept` or reload nftables _after_ Docker starts.
*   **5GHz Wi-Fi Not Visible:**  
    _Cause:_ Regulatory domain not set.  
    _Fix:_ `sudo iw reg set IN` then toggle radio off/on.
*   **USB Over-current on Boot:**  
    _Fix:_ Ensure high-quality power supply (3.0A+). Use powered USB hub if necessary.
*   **AdGuard Home DNS Failure:**  
    _Fix:_ Disable IPv6 upstreams in AGH config; bind to `0.0.0.0`.
