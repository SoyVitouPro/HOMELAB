# 🚀 Setup Guide

This guide walks through setting up the Home LAB from scratch — from installing the base OS to spinning up all services.

---

## Prerequisites

- A dedicated server (see [Infrastructure](infrastructure.md) for hardware specs).
- A USB drive (≥ 8 GB) for the Proxmox installer.
- A laptop/workstation connected to the same network.
- Basic familiarity with Linux CLI and Docker.

---

## Table of Contents

1. [Install Proxmox VE](#1-install-proxmox-ve)
2. [Initial Proxmox Configuration](#2-initial-proxmox-configuration)
3. [Create the ZFS Storage Pool](#3-create-the-zfs-storage-pool)
4. [Create the Docker Host VM](#4-create-the-docker-host-vm)
5. [Install Docker on the VM](#5-install-docker-on-the-vm)
6. [Set Up Nginx Proxy Manager](#6-set-up-nginx-proxy-manager)
7. [Set Up AdGuard Home](#7-set-up-adguard-home)
8. [Set Up WireGuard VPN](#8-set-up-wireguard-vpn)
9. [Deploy Remaining Services](#9-deploy-remaining-services)
10. [Set Up Monitoring Stack](#10-set-up-monitoring-stack)

---

## 1. Install Proxmox VE

1. Download the latest **Proxmox VE ISO** from [https://www.proxmox.com/downloads](https://www.proxmox.com/downloads).
2. Flash it to a USB drive with [Rufus](https://rufus.ie/) (Windows) or `dd` (Linux/macOS):
   ```bash
   # Linux/macOS – replace /dev/sdX with your USB device
   sudo dd if=proxmox-ve_*.iso of=/dev/sdX bs=4M status=progress && sync
   ```
3. Boot the server from the USB drive and follow the Proxmox installer:
   - **Target disk:** Select the SSD (256 GB boot drive).
   - **Country / Timezone:** Set your locale.
   - **Password & Email:** Set a strong root password and an admin email for alerts.
   - **Network:** Set a static IP (e.g., `192.168.1.10/24`), gateway (`192.168.1.1`), and DNS (`192.168.1.1`).
4. After installation, remove the USB drive and reboot.
5. Access the Proxmox web UI at `https://192.168.1.10:8006`.

---

## 2. Initial Proxmox Configuration

### Remove the Enterprise Repository (no subscription)

```bash
# SSH into Proxmox
ssh root@192.168.1.10

# Disable enterprise repo
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" \
  > /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list

# Update and upgrade
apt update && apt full-upgrade -y
```

### Install Useful Tools

```bash
apt install -y vim curl wget git htop net-tools
```

### Configure Email Alerts (optional)

```bash
apt install -y libsasl2-modules postfix-pcre

# Configure /etc/postfix/main.cf for your SMTP relay (e.g., Gmail, SendGrid)
# Then test with:
echo "Test email from Proxmox" | mail -s "Proxmox Alert Test" your@email.com
```

---

## 3. Create the ZFS Storage Pool

Run this on the Proxmox host after identifying your HDD device names (use `lsblk`):

```bash
# Create mirrored ZFS pool (replace sdb/sdc with your HDD device names)
zpool create -f data mirror /dev/sdb /dev/sdc

# Enable LZ4 compression (saves ~20–40% space with no real performance cost)
zfs set compression=lz4 data

# Set mount point
zfs set mountpoint=/mnt/data data

# Create child datasets
zfs create data/backups
zfs create data/media
zfs create data/nextcloud
zfs create data/vaultwarden
zfs create data/gitea
zfs create data/docker

# Verify
zpool status
zfs list
```

### Add the Pool to Proxmox Storage

In the Proxmox web UI:
1. Go to **Datacenter → Storage → Add → Directory**.
2. **ID:** `data`, **Directory:** `/mnt/data`, **Content:** ISO, Backup, Snippets.

---

## 4. Create the Docker Host VM

In the Proxmox web UI:

1. **Download Ubuntu Server ISO** (Proxmox → local storage → ISO Images → Download from URL):
   ```
   https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso
   ```
2. Click **Create VM** and configure:

   | Setting | Value |
   |---|---|
   | VM ID | `101` |
   | Name | `docker-host` |
   | OS | Ubuntu 24.04 ISO |
   | CPU | 4 cores (type: `host`) |
   | RAM | 16384 MB (16 GB), no balloon |
   | Disk | 64 GB on NVMe storage (`fast`) |
   | Network | `virtio`, bridge `vmbr0`, VLAN tag `1` |

3. Start the VM and follow the Ubuntu installer. Set:
   - Static IP: `192.168.1.20` (or assign via DHCP reservation).
   - Hostname: `docker-host`.
   - Enable OpenSSH server.
4. After installation, SSH in and update:
   ```bash
   ssh user@192.168.1.20
   sudo apt update && sudo apt upgrade -y && sudo reboot
   ```

---

## 5. Install Docker on the VM

```bash
# Install dependencies
sudo apt install -y ca-certificates curl gnupg

# Add Docker's GPG key and repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine + Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add your user to the docker group
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

### Install Loki Docker Log Driver (optional)

```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

Add to `/etc/docker/daemon.json` to make Loki the default log driver:

```json
{
  "log-driver": "loki",
  "log-opts": {
    "loki-url": "http://192.168.1.12:3100/loki/api/v1/push",
    "loki-pipeline-stages": ""
  }
}
```

---

## 6. Set Up Nginx Proxy Manager

```bash
mkdir -p /opt/docker/proxy && cd /opt/docker/proxy

cat > docker-compose.yml << 'EOF'
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
EOF

docker compose up -d
```

1. Open `http://192.168.1.20:81` in your browser.
2. Log in with `admin@example.com` / `changeme` and immediately change the credentials.
3. Add proxy hosts for each service (see [Services](services.md) for ports).

---

## 7. Set Up AdGuard Home

On **LXC 200** (created via Proxmox → Create CT, Debian 12, `192.168.1.11`):

```bash
# Download and run the AdGuard Home installer
curl -s -S -L https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

# AdGuard Home setup wizard is available at:
# http://192.168.1.11:3000
```

Post-setup checklist:
- [ ] Change admin username and password.
- [ ] Set upstream DNS to `https://dns.cloudflare.com/dns-query` (DoH).
- [ ] Add blocklists (AdGuard DNS filter, OISD, Steven Black Hosts).
- [ ] Add local DNS rewrites for `*.homelab.local` pointing to `192.168.1.20`.
- [ ] Set router DHCP to hand out `192.168.1.11` as DNS server for all VLANs.

---

## 8. Set Up WireGuard VPN

On **LXC 202** (Debian 12, `192.168.1.13`):

```bash
apt update && apt install -y wireguard

# Generate server key pair
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
chmod 600 /etc/wireguard/server_private.key

SERVER_PRIVATE=$(cat /etc/wireguard/server_private.key)

# Create server config
cat > /etc/wireguard/wg0.conf << EOF
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = ${SERVER_PRIVATE}

# Enable IP forwarding
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
EOF

# Enable IP forwarding
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p

# Start and enable WireGuard
systemctl enable --now wg-quick@wg0
```

Add a peer (repeat for each device):

```bash
# Generate peer key pair
wg genkey | tee peer_private.key | wg pubkey > peer_public.key
PEER_PUBLIC=$(cat peer_public.key)

# Add to server config
cat >> /etc/wireguard/wg0.conf << EOF

[Peer]
# Name: MyPhone
PublicKey = ${PEER_PUBLIC}
AllowedIPs = 10.8.0.2/32
EOF

wg syncconf wg0 <(wg-quick strip wg0)
```

Open port 51820/UDP on the OPNsense firewall and forward it to the WireGuard LXC.

---

## 9. Deploy Remaining Services

Each service has its own directory under `/opt/docker/`. Below is a template for deploying any service:

```bash
mkdir -p /opt/docker/<service-name> && cd /opt/docker/<service-name>
# Create docker-compose.yml and .env files
# Then:
docker compose up -d
# Add a proxy host in Nginx Proxy Manager
```

### Vaultwarden

```bash
mkdir -p /opt/docker/security && cd /opt/docker/security

cat > docker-compose.yml << 'EOF'
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      - SIGNUPS_ALLOWED=false
      - WEBSOCKET_ENABLED=true
    volumes:
      - /mnt/data/vaultwarden:/data
    ports:
      - "8080:80"
      - "3012:3012"
EOF

docker compose up -d
```

### Jellyfin

```bash
mkdir -p /opt/docker/media && cd /opt/docker/media

cat > docker-compose.yml << 'EOF'
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    volumes:
      - /mnt/data/media:/media
      - ./config:/config
      - ./cache:/cache
EOF

docker compose up -d
```

### Nextcloud

```bash
mkdir -p /opt/docker/storage && cd /opt/docker/storage

cat > .env << 'EOF'
MYSQL_ROOT_PASSWORD=change_me_root
MYSQL_PASSWORD=change_me_nc
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
EOF

cat > docker-compose.yml << 'EOF'
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    env_file: .env
    volumes:
      - ./db:/var/lib/mysql

  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    depends_on:
      - db
      - redis
    ports:
      - "8081:80"
    env_file: .env
    environment:
      - REDIS_HOST=redis
    volumes:
      - /mnt/data/nextcloud:/var/www/html/data
      - ./config:/var/www/html/config
EOF

docker compose up -d
```

### Gitea

```bash
mkdir -p /opt/docker/development && cd /opt/docker/development

cat > docker-compose.yml << 'EOF'
services:
  gitea:
    image: gitea/gitea:latest
    container_name: gitea
    restart: unless-stopped
    environment:
      - USER_UID=1000
      - USER_GID=1000
    ports:
      - "3000:3000"
      - "222:22"
    volumes:
      - /mnt/data/gitea:/data
EOF

docker compose up -d
```

---

## 10. Set Up Monitoring Stack

On **LXC 201** (Debian 12, `192.168.1.12`):

```bash
# Install Docker on the LXC
curl -fsSL https://get.docker.com | sh

mkdir -p /opt/monitoring && cd /opt/monitoring

cat > docker-compose.yml << 'EOF'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=change_me
    volumes:
      - grafana_data:/var/lib/grafana

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - loki_data:/loki

  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime_data:/app/data

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
  uptime_data:
EOF

# Create a minimal Prometheus config
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets:
          - '192.168.1.20:9100'   # Docker host VM
          - '192.168.1.11:9100'   # AdGuard LXC
          - '192.168.1.12:9100'   # Monitoring LXC
EOF

docker compose up -d
```

Post-setup checklist:
- [ ] Log into Grafana at `http://192.168.1.12:3000` (default: `admin` / `change_me`).
- [ ] Add Prometheus as a data source (`http://prometheus:9090`).
- [ ] Add Loki as a data source (`http://loki:3100`).
- [ ] Import dashboard ID **1860** (Node Exporter Full) from [grafana.com/dashboards](https://grafana.com/grafana/dashboards/).
- [ ] Configure Uptime Kuma monitors for all services.

---

## Maintenance

### Update all containers

```bash
# Pull latest images and recreate containers
cd /opt/docker/<service>
docker compose pull && docker compose up -d
```

### One-liner to update all stacks

```bash
for dir in /opt/docker/*/; do
  echo "Updating $dir..."
  docker compose -f "${dir}docker-compose.yml" pull --quiet
  docker compose -f "${dir}docker-compose.yml" up -d --remove-orphans
done

# Remove unused images after update
docker image prune -f
```

### Proxmox VM/LXC Snapshot (backup before major changes)

```bash
# On Proxmox host – snapshot VM 101 before updates
vzdump 101 --storage data --compress zstd --mode snapshot
```
