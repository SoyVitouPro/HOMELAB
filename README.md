# 🏠 HOMELAB

> A self-hosted home server setup running containerised services for privacy, learning, and fun.

## Overview

This repository documents the complete infrastructure, services, and setup procedures for a self-hosted Home LAB. The goal is to centralise services (media, monitoring, networking, automation, and more) on local hardware while keeping everything reproducible and easy to maintain.

---

## 📂 Documentation

| Document | Description |
|---|---|
| [Infrastructure](docs/infrastructure.md) | Hardware, network topology, VLAN design, and storage layout |
| [Services](docs/services.md) | Every service installed, its purpose, and the port it runs on |
| [Setup Guide](docs/setup.md) | Step-by-step installation and configuration walkthrough |

---

## 🗂️ Quick Reference

### Core Stack

| Category | Tool |
|---|---|
| Hypervisor / Container Runtime | Proxmox VE + Docker |
| Reverse Proxy | Nginx Proxy Manager |
| DNS / Ad-blocking | AdGuard Home |
| VPN | WireGuard |
| Monitoring | Grafana + Prometheus + Uptime Kuma |
| Container Management | Portainer |
| File Storage | Nextcloud |
| Media Server | Jellyfin |
| Password Manager | Vaultwarden |
| Git Server | Gitea |

### Network Summary

| Segment | Subnet | Purpose |
|---|---|---|
| Management | `192.168.1.0/24` | Servers, switches, and infrastructure devices |
| IoT | `192.168.10.0/24` | Smart home and IoT devices (isolated) |
| Trusted | `192.168.20.0/24` | Personal laptops and phones |
| Guest | `192.168.30.0/24` | Guest Wi-Fi (internet-only) |

---

## 🚀 Getting Started

1. Read [Infrastructure](docs/infrastructure.md) to understand the hardware and network design.
2. Read [Services](docs/services.md) to see what is installed and why.
3. Follow the [Setup Guide](docs/setup.md) to reproduce the environment from scratch.

---

## 📝 License

[MIT](LICENSE)
