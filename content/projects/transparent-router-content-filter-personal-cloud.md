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

**System Role:** A high-availability Raspberry Pi 4 hosting a redundant RAID 1 storage array, acting as a transparent router for a LAN subnet, enforcing DNS filtering via AdGuard Home, and serving a Nextcloud instance externally via Cloudflare Tunnels.


**Host:** Raspberry Pi 4 Model B (4GB RAM)

**Hostname:** `pi03`

**OS:** Debian 12 (Bookworm)

**Functions:** Multi-WAN Router, AdGuard Home DNS Filter, Nextcloud Storage (RAID 1), Cloudflare Tunnel.

**Date:** November 2025

System Architecture & Network Design
----------------------------------------

This system operates as a transparent router for a LAN subnet while hosting secure personal cloud services accessible from the internet.

### Network Interfaces & Zones

| Interface | Name (Alias) | Type | Role | Subnet/IP | Metric |
| --- | --- | --- | --- | --- | --- |
| **eth0** | `eth0-mgmt` | Built-in | Management (SSH) | DHCP (No Default GW) | 102 |
| **eth1** | **wan0** | USB NIC | **Primary WAN** (Static) | <WAN0\_STATIC\_IP> | 100 |
| **eth2** | **lan0** | USB NIC | **LAN Gateway** | `192.168.10.1/24` | N/A |
| **wlan0** | **wan1** | Built-in Wi-Fi | **Backup WAN** (DHCP) | DHCP (Metric 200) | 200 |
| **docker0** | `docker0` | Bridge | Docker Default | `172.17.0.1/16` | N/A |
| **br-xx** | `br-xxxx` | Bridge | Compose Network | `172.x.0.1/16` | N/A |

- - -

1\. Storage Layer: RAID 1 Configuration
---------------------------------------

The foundation of the system is a mirrored RAID 1 array using two external USB drives. This setup provides redundancy against single-drive failure.

### 1.1 Partitioning (gdisk)

The disks were partitioned with type code `fd00` (Linux RAID) to utilize the entire available space.

    # Install tools
    sudo apt update && sudo apt install mdadm gdisk
    
    # Partitioning (Repeat for both /dev/sda and /dev/sdb)
    sudo gdisk /dev/sda
    # Command sequence: 
    # n (new), 1 (partition number), enter (default start), enter (default end), fd00 (type), w (write)

### 1.2 Array Creation

Creating the MDADM array mirroring `/dev/sda1` and `/dev/sdb1`.

    sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1

### 1.3 Filesystem & Mounting

The array is formatted as ext4 and mounted persistently.

    # Format
    sudo mkfs.ext4 -F /dev/md0
    
    # Create Mount Point
    sudo mkdir -p /mnt/cloudstorage
    
    # Get UUID for fstab
    sudo blkid /dev/md0

/etc/fstab

    # Add the following line (Replace UUID):
    UUID=<YOUR_RAID_UUID> /mnt/cloudstorage ext4 defaults,nofail 0 2

### 1.4 Monitoring

Configured email alerts for disk failure events.

/etc/mdadm/mdadm.conf

    MAILADDR your_email@example.com
    MAILFROM pi03-raid-monitor

    # Enable monitoring service
    sudo systemctl enable mdadm.service
    sudo systemctl start mdadm.service

* * *

2\. Network Layer: Physical & Logical
-------------------------------------

This section covers the complex USB timing fixes and persistent naming required to make a Raspberry Pi a stable multi-NIC router.

### 2.1 Hardware Timing Fix (The "USB Race Condition")

**Critical Historical Context:** The external USB NIC (wan0) failed to initialize before NetworkManager attempted to configure it during boot. We implemented a `udev` rule to force a 10-second delay and a re-apply trigger.

/etc/udev/rules.d/70-net-delay.rules

    # Force a 10-second delay for specific USB NIC hardware (wan0)
    # Replace ATTRS{devpath} with the stable physical USB path (e.g., 1.1)
    SUBSYSTEM=="net", ACTION=="add", ATTRS{devpath}=="1.1", NAME="wan0", RUN+="/bin/sh -c 'sleep 10 > /dev/null && /usr/bin/nmcli device reapply wan0'"

### 2.2 Persistent Interface Naming

To prevent interface names (eth1, eth2) from swapping on reboot, we bound them to physical paths.

/etc/udev/rules.d/70-persistent-net.rules

    # Static WAN (USB NIC) - Locked to USB Port Path 1.1
    SUBSYSTEM=="net", ACTION=="add", ATTRS{devpath}=="1.1", NAME="wan0"
    
    # LAN Gateway (USB NIC) - Locked to USB Port Path 2.4
    SUBSYSTEM=="net", ACTION=="add", ATTRS{devpath}=="2.4", NAME="lan0"
    
    # Backup WAN (Internal Wi-Fi) - Locked to MAC Address
    SUBSYSTEM=="net", ACTION=="add", ATTR{address}=="dc:a6:32:xx:xx:xx", NAME="wan1"

### 2.3 NetworkManager Profiles (Multi-WAN)

#### Primary WAN (wan0) - Static IP

    nmcli connection add type ethernet con-name wan0-wan-static ifname wan0 \
      ipv4.method manual ipv4.addresses 172.29.72.195/26 ipv4.gateway 172.29.72.193 \
      ipv4.dns "8.8.8.8 4.2.2.1" ipv4.route-metric 100

#### Backup WAN (wan1) - Wi-Fi DHCP

_Note: Regulatory domain set via `sudo iw reg set IN` to enable 5GHz._

    nmcli connection add type wifi con-name wan1-wifi-dhcp ifname wan1 \
      ssid "SSID_NAME" wifi-sec.key-mgmt wpa-psk wifi-sec.psk "PASSWORD" \
      ipv4.method auto ipv4.route-metric 200 connection.interface-name wan1

#### LAN Gateway (lan0) - Static IP

    nmcli connection add type ethernet con-name lan0-lan ifname lan0 \
      ipv4.method manual ipv4.addresses 192.168.100.1/24 ipv4.never-default yes

* * *

3\. Firewall & Routing: Nftables & Docker Integration
-----------------------------------------------------

The core routing logic. This configuration required significant debugging to handle the race condition between Docker networking and the firewall service.

### 3.1 The "Interface Not Found" Race Condition

**Historical Issue:** On boot, `nftables.service` failed because it tried to apply rules for `docker0` and `br-xxxx` before Docker had created them.  
**Fix:** A systemd override was created to force dependency on Docker.

/etc/systemd/system/nftables.service.d/override.conf

    [Unit]
    # Wait for Network interfaces and IP Forwarding
    After=network-online.target sysctl.service
    Wants=network-online.target sysctl.service
    
    # CRITICAL: Wait for Docker to start and create bridges
    Requires=docker.service
    After=docker.service

### 3.2 Firewall Ruleset

/etc/nftables.conf

    #!/usr/sbin/nft -f
    flush ruleset
    
    table ip filter {
        chain input {
            type filter hook input priority 0; policy drop;
            ct state established,related accept
            ip protocol icmp accept
            iif lo accept
            
            # SSH Access
            iif eth0 tcp dport 22 accept
            
            # LAN Services (DNS, DHCP, Web UI)
            iif lan0 udp dport { 53, 67 } accept
            iif lan0 tcp dport { 53, 8080, 8081 } accept
        }
    
        chain forward {
            type filter hook forward priority 0; policy drop;
            ct state established,related accept
            
            # 1. LAN to WAN (New Connections)
            iif lan0 oif wan0 ct state new accept
            iif lan0 oif wan1 ct state new accept
            
            # 2. Docker to WAN (New Connections)
            # Explicitly allows containers to reach the internet
            iif docker0 oif wan0 accept
            iif docker0 oif wan1 accept
            # Replace with your specific dynamic bridge ID
            iif br-f9536f6adf36-CHANGEME oif wan0 accept
            iif br-f9536f6adf36-CHANGEME oif wan1 accept
            
            # 3. Intra-LAN
            iif lan0 oif lan0 accept
        }
    }
    
    table ip nat {
        chain postrouting {
            type nat hook postrouting priority 100; policy accept;
            # Masquerade (NAT) for both WAN interfaces
            oif wan0 masquerade
            oif wan1 masquerade
        }
        
        chain prerouting {
            type nat hook prerouting priority -100; policy accept;
            # Transparent DNS Redirect (Force LAN to AdGuard Home)
            iif lan0 udp dport 53 redirect
            iif lan0 tcp dport 53 redirect
        }
    }

### 3.3 Enabling Services

    # Enable IP Forwarding persistently
    echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-ip-forward.conf
    
    # Enable Firewall
    sudo systemctl enable nftables
    sudo systemctl start nftables

* * *

4\. Core Services: DHCP & DNS
-----------------------------

### 4.1 Dnsmasq (DHCP)

Serves IPs to the LAN and forces clients to use the Pi's IP for DNS.

/etc/dnsmasq.conf

    interface=lan0
    dhcp-range=192.168.10.100,192.168.10.250,255.255.255.0,12h
    dhcp-option=3,192.168.10.1   # Default Gateway
    dhcp-option=6,192.168.10.1   # DNS Server
    port=0                       # Disable built-in DNS (AdGuard handles this)

    sudo systemctl enable dnsmasq
    sudo systemctl restart dnsmasq

### 4.2 AdGuard Home (DNS Filter)

**Critical Fixes:** We encountered upstream timeout issues. The fix was disabling IPv6 upstreams and binding to all interfaces to handle the local redirection.

-   **Binding:** `0.0.0.0:53` (Must listen on all interfaces).
-   **Upstreams:** `8.8.8.8`, `1.1.1.1` (IPv4 Only).
-   **Config:** `disable_ipv6: true`.

* * *

5\. Application Layer: Nextcloud & Cloudflare
---------------------------------------------

Hosted in `~/nextcloud-docker/` using Docker Compose.

### 5.1 The MariaDB "Access Denied" Fix

**Historical Issue:** The official MariaDB container failed to create the user correctly via environment variables on this architecture, leading to an infinite "Access Denied" loop on boot.  
**Solution:** We disabled env-var user creation and used a manual SQL init script.

~/nextcloud-docker/init-db/init.sql

    -- Allow connection from ANY Docker host (%)
    CREATE USER 'nextclouduser'@'%' IDENTIFIED BY 'YOUR_SECURE_PASSWORD';
    GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'%';
    FLUSH PRIVILEGES;

### 5.2 Docker Compose Configuration

~/nextcloud-docker/docker-compose.yml

    version: '3.7'
    services:
      db:
        image: mariadb:10.6
        restart: always
        command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
        volumes:
          - ./db:/var/lib/mysql
          - ./init-db:/docker-entrypoint-initdb.d # Custom SQL Fix
        environment:
          MYSQL_ROOT_PASSWORD: ROOT_PASSWORD
          MYSQL_DATABASE: nextcloud
          # Removed MYSQL_USER/PASSWORD to prevent conflict with init.sql
    
      app:
        image: nextcloud
        restart: always
        ports:
          - 8081:80
        volumes:
          - /mnt/cloudstorage/nextcloud_data:/var/www/html/data
          - ./config:/var/www/html/config
        environment:
          MYSQL_HOST: db
          MYSQL_DATABASE: nextcloud
          MYSQL_USER: nextclouduser # Matches init.sql
          MYSQL_PASSWORD: YOUR_SECURE_PASSWORD
          NEXTCLOUD_TRUSTED_DOMAINS: "cloud.yourdomain.com"

### 5.3 Cloudflare Tunnel

Exposes the service securely without opening ports.

    cloudflared tunnel create pi03-cloud
    cloudflared tunnel route dns pi03-cloud cloud.yourdomain.com

**Config:** Ingress points to `http://localhost:8081`.

* * *

6\. Maintenance & Troubleshooting
---------------------------------

### System Recovery

-   **Restart Network Stack:**
    
        sudo systemctl restart NetworkManager
        sudo systemctl restart nftables
        sudo systemctl restart dnsmasq
    
-   **Reload Udev Rules:** `sudo udevadm control --reload-rules`

### RAID Management

-   **Check Status:** `cat /proc/mdstat`
-   **Detail View:** `sudo mdadm --detail /dev/md0`
-   **Test Email:** `sudo mdadm --monitor --scan --test`

### Logs to Watch

-   **Firewall:** `sudo systemctl status nftables`
-   **DHCP:** `journalctl -u dnsmasq -f`
-   **Docker Database:** `docker logs nextcloud-docker_db_1`


