# 🏗️ Infrastructure

This document describes the physical hardware, network topology, VLAN design, and storage layout of the Home LAB.

---

## Hardware

### Primary Server

| Component | Specification |
|---|---|
| **Form Factor** | Mini-ITX / Tower |
| **CPU** | Intel Core i5 / AMD Ryzen 5 (6+ cores recommended) |
| **RAM** | 32 GB DDR4 ECC (minimum 16 GB) |
| **Boot Drive** | 256 GB SSD (OS + Proxmox) |
| **Data Drive 1** | 2 × 4 TB HDD in ZFS mirror (bulk storage) |
| **Data Drive 2** | 1 TB NVMe SSD (fast VM disks / databases) |
| **NIC** | 2 × 1 GbE (management + trunk) |
| **Power Supply** | 80+ Gold, 450 W |

### Network Hardware

| Device | Role |
|---|---|
| Router / Firewall | OPNsense on dedicated mini-PC (or Raspberry Pi 4 with Pi-hole only) |
| Managed Switch | TP-Link TL-SG108E (8-port, VLAN-capable) |
| Wi-Fi Access Point | TP-Link EAP245 (or any OpenWrt-compatible AP) |
| Raspberry Pi 4 | Secondary DNS + network monitoring (optional) |

---

## Network Topology

```
Internet
    │
    ▼
[ISP Modem / ONT]
    │  (WAN)
    ▼
[OPNsense Firewall / Router]
    │  (LAN trunk – 802.1Q)
    ▼
[Managed Switch]
    ├── VLAN 1  (Management)  ── Home LAB Server, Switch, AP
    ├── VLAN 10 (IoT)         ── Smart plugs, cameras, sensors
    ├── VLAN 20 (Trusted)     ── Personal laptops, phones
    └── VLAN 30 (Guest)       ── Guest Wi-Fi (NAT, internet-only)
```

---

## VLAN Design

| VLAN ID | Name | Subnet | Gateway | DHCP Range | Purpose |
|---|---|---|---|---|---|
| 1 | Management | `192.168.1.0/24` | `192.168.1.1` | `.100 – .200` | Infrastructure management |
| 10 | IoT | `192.168.10.0/24` | `192.168.10.1` | `.100 – .200` | Isolated IoT devices |
| 20 | Trusted | `192.168.20.0/24` | `192.168.20.1` | `.100 – .200` | Personal devices |
| 30 | Guest | `192.168.30.0/24` | `192.168.30.1` | `.100 – .200` | Internet-only guest access |

### Firewall Rules Summary

| From → To | Allowed |
|---|---|
| Trusted → Management | ✅ Yes (full access to Lab services) |
| Trusted → IoT | ✅ Yes (to control devices) |
| IoT → Trusted | ❌ No (IoT cannot initiate connections) |
| IoT → Management | ❌ No |
| Guest → Trusted | ❌ No |
| Guest → Management | ❌ No |
| Guest → Internet | ✅ Yes |

---

## Static IP Assignments (Management VLAN)

| IP Address | Hostname | Role |
|---|---|---|
| `192.168.1.1` | `gateway` | OPNsense router |
| `192.168.1.2` | `switch` | Managed switch |
| `192.168.1.3` | `ap` | Wi-Fi access point |
| `192.168.1.10` | `homelab` | Primary server (Proxmox VE host) |
| `192.168.1.11` | `adguard` | LXC 200 – AdGuard Home DNS |
| `192.168.1.12` | `monitoring` | LXC 201 – Grafana / Prometheus |
| `192.168.1.13` | `vpn` | LXC 202 – WireGuard VPN |
| `192.168.1.20` | `docker-host` | VM 101 – Docker application host |

---

## Virtualisation Layout (Proxmox VE)

All workloads run under **Proxmox VE**. LXC containers are used for lightweight services; full VMs are used for anything that needs kernel-level isolation.

```
Proxmox VE  (192.168.1.10)
├── VM 100  – OPNsense (firewall/router)          [2 vCPU, 2 GB RAM]
├── VM 101  – Docker Host (main services)          [4 vCPU, 16 GB RAM]
│    ├── Nginx Proxy Manager
│    ├── Portainer
│    ├── Nextcloud
│    ├── Vaultwarden
│    ├── Gitea
│    ├── Jellyfin
│    ├── Uptime Kuma
│    └── ...
├── LXC 200 – AdGuard Home (DNS)                   [1 vCPU, 512 MB RAM]
├── LXC 201 – Grafana + Prometheus + Loki          [2 vCPU, 2 GB RAM]
└── LXC 202 – WireGuard VPN                        [1 vCPU, 512 MB RAM]
```

---

## Storage Layout

| Mount Point | Type | Size | Use |
|---|---|---|---|
| `/` (Proxmox OS) | SSD | 256 GB | Proxmox OS boot drive |
| `/mnt/fast` | NVMe | 1 TB | VM disks, databases |
| `/mnt/data` | ZFS mirror (2 × HDD) | ~3.6 TB usable | Media, backups, Nextcloud data |

### ZFS Pool Setup

```bash
# Create mirrored ZFS pool for bulk data
zpool create data mirror /dev/sdb /dev/sdc

# Enable compression and set mount point
zfs set compression=lz4 data
zfs set mountpoint=/mnt/data data
```

---

## Backups

| What | Where | How Often |
|---|---|---|
| Proxmox VM/LXC snapshots | Local ZFS dataset `/mnt/data/backups` | Daily |
| Nextcloud data | External USB drive or cloud (rclone to B2/S3) | Weekly |
| Gitea repositories | Local ZFS + offsite rclone | Daily |
| Config files | This Git repository | On change |

> **Tip:** Follow the **3-2-1 backup rule** — 3 copies, 2 different media, 1 offsite.
