# 🛠️ Services

This document lists every service installed in the Home LAB, its purpose, the port(s) it exposes, and how it is deployed.

All services run as **Docker containers** on the Docker Host VM (`192.168.1.10`, VM 101) unless stated otherwise.  
Each service is accessible via the internal domain `*.homelab.local` through Nginx Proxy Manager.

---

## Service Index

| # | Service | Category | Port(s) | URL |
|---|---|---|---|---|
| 1 | [Nginx Proxy Manager](#1-nginx-proxy-manager) | Networking | 80, 443, 81 | `proxy.homelab.local` |
| 2 | [AdGuard Home](#2-adguard-home) | Networking | 53, 3000, 80 | `adguard.homelab.local` |
| 3 | [WireGuard](#3-wireguard) | Networking | 51820/UDP | — |
| 4 | [Portainer](#4-portainer) | Management | 9000, 9443 | `portainer.homelab.local` |
| 5 | [Grafana](#5-grafana) | Monitoring | 3000 | `grafana.homelab.local` |
| 6 | [Prometheus](#6-prometheus) | Monitoring | 9090 | `prometheus.homelab.local` |
| 7 | [Loki](#7-loki) | Monitoring | 3100 | (internal only) |
| 8 | [Uptime Kuma](#8-uptime-kuma) | Monitoring | 3001 | `status.homelab.local` |
| 9 | [Nextcloud](#9-nextcloud) | Storage | 80, 443 | `cloud.homelab.local` |
| 10 | [Jellyfin](#10-jellyfin) | Media | 8096, 8920 | `media.homelab.local` |
| 11 | [Vaultwarden](#11-vaultwarden) | Security | 80 | `vault.homelab.local` |
| 12 | [Gitea](#12-gitea) | Development | 3000, 22 | `git.homelab.local` |
| 13 | [Homepage](#13-homepage) | Dashboard | 3000 | `home.homelab.local` |

---

## Networking

### 1. Nginx Proxy Manager

**Purpose:** Reverse proxy that terminates HTTPS for all internal services. Provides a simple web UI to manage proxy hosts, SSL certificates (Let's Encrypt via DNS challenge), and access lists.

**Image:** `jc21/nginx-proxy-manager:latest`

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 80 | TCP | HTTP (redirect to HTTPS) |
| 443 | TCP | HTTPS proxy |
| 81 | TCP | Admin web UI |

**Key Config:**
- SSL certificates issued via Let's Encrypt DNS challenge (no need to open port 443 externally).
- All services behind `*.homelab.local` use a self-signed or local CA cert for LAN-only access.

---

### 2. AdGuard Home

**Purpose:** Network-wide DNS server and ad/tracker blocker. Replaces Pi-hole. Provides DNS over HTTPS (DoH) and DNS over TLS (DoT).

**Deployed on:** LXC 200 (`192.168.1.11`)

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 53 | TCP/UDP | DNS queries |
| 80 | TCP | Web UI (HTTP) |
| 3000 | TCP | Initial setup UI |
| 853 | TCP | DNS over TLS |

**Key Config:**
- Upstream DNS: Cloudflare DoH (`https://dns.cloudflare.com/dns-query`)
- Blocklists: AdGuard DNS filter, OISD, Steven Black Hosts.
- Local DNS rewrites for `*.homelab.local`.

---

### 3. WireGuard

**Purpose:** Fast, modern VPN to securely access the Home LAB remotely.

**Deployed on:** LXC 202

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 51820 | UDP | VPN tunnel |

**Key Config:**
- Server subnet: `10.8.0.0/24`.
- Peer DNS: AdGuard Home (`192.168.1.11`) so ad-blocking applies on VPN too.
- Split tunnelling configured so only `192.168.0.0/16` traffic goes through the VPN.

---

## Management

### 4. Portainer

**Purpose:** Web UI for managing Docker containers, stacks (Docker Compose), images, volumes, and networks.

**Image:** `portainer/portainer-ce:latest`

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 9000 | TCP | Web UI (HTTP) |
| 9443 | TCP | Web UI (HTTPS) |
| 8000 | TCP | Edge agent tunnel |

---

## Monitoring

### 5. Grafana

**Purpose:** Visualisation and dashboarding platform. Displays metrics from Prometheus and logs from Loki.

**Image:** `grafana/grafana:latest`

**Deployed on:** LXC 201

**Port:** `3000`

**Dashboards installed:**
- Node Exporter Full (system metrics)
- Docker container stats
- AdGuard Home stats
- Proxmox VE metrics

---

### 6. Prometheus

**Purpose:** Time-series metrics collection. Scrapes exporters running on all hosts.

**Image:** `prom/prometheus:latest`

**Deployed on:** LXC 201

**Port:** `9090`

**Exporters scraped:**

| Exporter | Target | Metrics |
|---|---|---|
| `node_exporter` | All Linux hosts | CPU, RAM, disk, network |
| `cadvisor` | Docker host | Container resource usage |
| `proxmox_exporter` | Proxmox VE | VM/LXC resource usage |
| `adguard_exporter` | AdGuard Home | DNS query stats |

---

### 7. Loki

**Purpose:** Log aggregation. Collects logs from all Docker containers via the `loki-docker-driver` log plugin and makes them searchable in Grafana.

**Image:** `grafana/loki:latest`

**Deployed on:** LXC 201

**Port:** `3100` (internal only)

---

### 8. Uptime Kuma

**Purpose:** Self-hosted uptime monitoring tool. Sends alerts (Telegram, email, etc.) when a service goes down.

**Image:** `louislam/uptime-kuma:latest`

**Port:** `3001`

**Monitors configured:**
- All `*.homelab.local` services (HTTP/HTTPS)
- External internet check (ping / HTTP)
- WireGuard VPN endpoint

---

## Storage

### 9. Nextcloud

**Purpose:** Self-hosted cloud storage and collaboration platform (alternative to Google Drive / OneDrive). Features: file sync, calendar, contacts, notes.

**Image:** `nextcloud:latest` (with `nextcloud-aio` recommended for production)

**Port:** `80` (behind Nginx Proxy Manager → HTTPS)

**Data path:** `/mnt/data/nextcloud`

**Integrations:**
- Collabora Online (LibreOffice in the browser) via Nextcloud Office app.
- Talk (video/audio calls).

---

## Media

### 10. Jellyfin

**Purpose:** Free, open-source media server for streaming movies, TV shows, and music. Alternative to Plex with no subscription required.

**Image:** `jellyfin/jellyfin:latest`

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 8096 | TCP | Web UI / clients (HTTP) |
| 8920 | TCP | Web UI / clients (HTTPS) |
| 7359 | UDP | Auto-discovery (LAN) |
| 1900 | UDP | DLNA (optional) |

**Media paths:**

| Path | Content |
|---|---|
| `/mnt/data/media/movies` | Movies |
| `/mnt/data/media/tvshows` | TV series |
| `/mnt/data/media/music` | Music library |

---

## Security

### 11. Vaultwarden

**Purpose:** Lightweight, unofficial Bitwarden-compatible server. Stores and syncs passwords across all devices using the official Bitwarden apps and browser extensions.

**Image:** `vaultwarden/server:latest`

**Port:** `80` (behind Nginx Proxy Manager → HTTPS — **HTTPS is required** by Bitwarden clients)

**Data path:** `/mnt/data/vaultwarden`

> ⚠️ Keep `SIGNUPS_ALLOWED=false` after creating your account to prevent unauthorized registrations.

---

## Development

### 12. Gitea

**Purpose:** Lightweight, self-hosted Git service. Alternative to GitHub/GitLab. Stores personal and Home LAB configuration repositories.

**Image:** `gitea/gitea:latest`

**Ports:**

| Port | Protocol | Use |
|---|---|---|
| 3000 | TCP | Web UI |
| 22 | TCP | SSH Git access |

**Data path:** `/mnt/data/gitea`

---

## Dashboard

### 13. Homepage

**Purpose:** A modern, highly customisable application dashboard. Displays live service status, system stats, and quick links to all services.

**Image:** `ghcr.io/gethomepage/homepage:latest`

**Port:** `3000` (behind Nginx Proxy Manager)

**Integrations:** Portainer, Proxmox, Jellyfin, AdGuard Home, Uptime Kuma, Nextcloud, Gitea.

---

## Docker Compose Structure

All services are organised into logical stacks under `/opt/docker/`:

```
/opt/docker/
├── proxy/          # Nginx Proxy Manager
├── monitoring/     # Grafana, Prometheus, Loki, Uptime Kuma
├── storage/        # Nextcloud (+ MariaDB, Redis)
├── media/          # Jellyfin
├── security/       # Vaultwarden
├── development/    # Gitea
└── dashboard/      # Homepage
```

Each directory contains a `docker-compose.yml` and an `.env` file for environment-specific variables (never committed to Git).
