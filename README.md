# 🏠 Home Server Setup Guide

A complete guide to my self-hosted infrastructure running on Proxmox with Docker/LXC containers. This repo documents the full stack from bare metal to running services so you can replicate or adapt it for your own setup.

---

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Hardware & OS](#hardware--os)
- [Proxmox Setup](#proxmox-setup)
- [LXC Containers](#lxc-containers)
  - [Portainer LXC (Docker Host)](#portainer-lxc-docker-host)
  - [Services LXC (Web/ML/Dev)](#services-lxc-webmldev)
- [Docker Stacks](#docker-stacks)
  - [Immich (Photo Management)](#immich-photo-management)
  - [Karakeep (Bookmark Manager)](#karakeep-bookmark-manager)
  - [Navidrome + Filebrowser (Music)](#navidrome--filebrowser-music)
  - [Paperless-NGX + AFFiNE (Documents & Notes)](#paperless-ngx--affine-documents--notes)
  - [RoMM (ROM Manager)](#romm-rom-manager)
  - [Nginx Proxy Manager](#nginx-proxy-manager)
- [Standalone Containers](#standalone-containers)
- [Networking](#networking)
- [Storage Layout](#storage-layout)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Physical Server                        │
│                   Proxmox VE 8.x                         │
│                                                           │
│  ┌──────────────────────┐  ┌──────────────────────────┐  │
│  │   LXC: Portainer     │  │   LXC: Services          │  │
│  │   (Alpine Linux)     │  │   (Alpine Linux)         │  │
│  │                      │  │                          │  │
│  │  Docker Engine       │  │  Nginx                   │  │
│  │  Portainer CE        │  │  VS Code Server          │  │
│  │  ~30 Containers      │  │  ML Models               │  │
│  │                      │  │  Websites                │  │
│  └──────────────────────┘  └──────────────────────────┘  │
│                                                           │
│  ┌──────────────────────────────────────────────────┐    │
│  │              Shared Storage (NFS/Bind)            │    │
│  │   /media/music  /media/paperless  /mnt/storage    │    │
│  └──────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**Docker Stacks running in Portainer:**

| Stack | Services |
|-------|----------|
| `immich` | immich-server, immich-ml, postgres, redis |
| `karakeep` | web, chrome, meilisearch |
| `navidrome-filebrowser` | navidrome, filebrowser |
| `paperless-ngx` | paperless, redis, postgres, affine, affine_redis |
| `romm` | romm, romm-db (mariadb) |
| `nginx-proxy-manager` | nginx-proxy-manager |

**Standalone containers:** pihole, homeassistant, homepage, glance, grocy, vaultwarden, tailscale, cloudflared, portainer

---

## Hardware & OS

> Adapt these steps to your specific hardware. The guide assumes a dedicated x86_64 machine.

**Recommended minimum specs:**
- CPU: 4+ cores (8+ recommended for ML workloads)
- RAM: 16 GB (32 GB recommended)
- Storage: 1× SSD for OS/apps (250 GB+), 1× HDD/SSD for media (1 TB+)
- Network: Gigabit Ethernet

---

## Proxmox Setup

### 1. Download & Flash

```bash
# Download Proxmox VE ISO from https://www.proxmox.com/en/downloads
# Flash to USB with Balena Etcher or Rufus
```

### 2. Install Proxmox VE

1. Boot from the USB drive
2. Select **Install Proxmox VE (Graphical)**
3. Accept EULA → select target disk
4. Set country, timezone, keyboard layout
5. Set a strong root password and email
6. Configure network:
   - Set a **static IP** for the management interface (e.g. `192.168.1.10/24`)
   - Set gateway and DNS
7. Complete installation and reboot

### 3. Post-Install Configuration

Access the web UI at `https://<your-ip>:8006` and open the shell, or SSH in:

```bash
ssh root@192.168.1.10
```

**Disable the enterprise repo and enable free community repo:**

```bash
# Disable enterprise subscription nag
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# Enable no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-community.list

# Update
apt update && apt dist-upgrade -y
```

**Disable Ceph repo (if not using Ceph):**

```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
```

### 4. Configure Storage

In the Proxmox web UI:
- **Datacenter → Storage → Add → Directory** for media drives
- For NVMe/SSD, the default `local-lvm` is used for VM/CT disks
- Passthrough a disk directly to an LXC if needed (add to `/etc/pve/lxc/<id>.conf`):

```
mp0: /dev/sdb,mp=/mnt/storage
```

---

## LXC Containers

### Portainer LXC (Docker Host)

This container runs Docker Engine and hosts all Docker stacks via Portainer.

#### Create the LXC

In Proxmox web UI: **Create CT**

| Setting | Value |
|---------|-------|
| OS Template | Alpine Linux 3.19 |
| Disk | 32 GB (local-lvm) |
| CPU | 4 cores |
| RAM | 8192 MB |
| Swap | 2048 MB |
| Network | DHCP or static IP (e.g. `192.168.1.20`) |

> **Important:** In the Options tab, enable **Nesting** (required for Docker inside LXC).

Via CLI on the Proxmox host, also add these to `/etc/pve/lxc/<CTID>.conf`:

```
features: keyctl=1,nesting=1
```

#### Configure Alpine & Install Docker

```bash
# Start the container and open its console
pct start <CTID>
pct enter <CTID>

# Update packages
apk update && apk upgrade

# Install Docker
apk add docker docker-compose-plugin curl bash

# Enable Docker on boot
rc-update add docker boot
service docker start

# Verify
docker --version
docker compose version
```

#### Install Portainer CE

```bash
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access Portainer at `https://<LXC-IP>:9443` and create your admin account.

---

### Services LXC (Web/ML/Dev)

This container runs Nginx, VS Code Server, ML models, and any non-Docker services.

#### Create the LXC

| Setting | Value |
|---------|-------|
| OS Template | Alpine Linux 3.19 |
| Disk | 64 GB (local-lvm) |
| CPU | 4 cores |
| RAM | 4096 MB |
| Network | Static IP (e.g. `192.168.1.21`) |

#### Install Core Services

```bash
pct enter <CTID>

apk update && apk upgrade

# Nginx
apk add nginx
rc-update add nginx boot
service nginx start

# Node.js (for VS Code Server and web apps)
apk add nodejs npm

# Python + pip (for ML)
apk add python3 py3-pip

# VS Code Server (code-server)
curl -fsSL https://code-server.dev/install.sh | sh
```

**Configure code-server as a service:**

```bash
# Create config
mkdir -p ~/.config/code-server
cat > ~/.config/code-server/config.yaml << EOF
bind-addr: 0.0.0.0:8443
auth: password
password: your-strong-password-here
cert: false
EOF

# Enable and start
rc-update add code-server boot
service code-server start
```

---

## Docker Stacks

All stacks are deployed via Portainer. To deploy a stack:
1. In Portainer: **Stacks → Add Stack**
2. Paste the compose file content
3. Add environment variables in the **Environment variables** section
4. Click **Deploy the stack**

> **Security note:** Never commit `.env` files or compose files containing real credentials to Git. Use environment variable substitution (`${VAR}`) and keep secrets in Portainer's environment variable UI or a `.env` file that is `.gitignore`d.

---

### Immich (Photo Management)

Self-hosted Google Photos alternative with ML face recognition and smart albums.

> Immich maintains its own official compose file. Always pull the latest from the [official repo](https://github.com/immich-app/immich).

```bash
# Download official compose and env template
curl -o docker-compose.yml \
  https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml

curl -o .env \
  https://github.com/immich-app/immich/releases/latest/download/example.env
```

**Key services in the stack:**
- `immich_server` — Main API & web UI (port 2283)
- `immich_machine_learning` — Face recognition, CLIP embeddings
- `immich_postgres` — PostgreSQL with pgvector extension
- `immich_redis` — Caching and job queues

Access at `http://<host>:2283`

---

### Karakeep (Bookmark Manager)

AI-powered bookmark and read-later manager with full-text search.

📄 Compose file: [`stacks/karakeep/docker-compose.yml`](stacks/karakeep/docker-compose.yml)

**Environment variables needed (create in Portainer or `.env` file):**

```env
NEXTAUTH_SECRET=<generate with: openssl rand -base64 32>
MEILI_MASTER_KEY=<generate with: openssl rand -base64 32>
NEXTAUTH_URL=http://<your-server-ip>:3469
# Optional: OPENAI_API_KEY=<your-key>  # for AI auto-tagging
```

**Services:**
- `karakeep-web` — Main web app (port 3469)
- `karakeep-chrome` — Headless Chrome for page snapshots
- `karakeep-meilisearch` — Full-text search engine

Access at `http://<host>:3469`

---

### Navidrome + Filebrowser (Music)

Navidrome is a self-hosted music server (Subsonic-compatible). Filebrowser provides a web UI to manage your music files.

📄 Compose file: [`stacks/navidrome-filebrowser/docker-compose.yml`](stacks/navidrome-filebrowser/docker-compose.yml)

**Volume mounts to update before deploying:**

| Path | Description |
|------|-------------|
| `/media/music` | Your music library directory |
| `/mnt/storage/appdata/navidrome` | Navidrome database/config |
| `/mnt/storage/appdata/filebrowser` | Filebrowser config |

**Services:**
- `navidrome` — Music streaming server (port 4533)
- `filebrowser` — Web file manager for music library (port 8088)

Access Navidrome at `http://<host>:4533` and connect any Subsonic-compatible app (Symfonium, Finamp, DSub).

---

### Paperless-NGX + AFFiNE (Documents & Notes)

Combined stack for document management and collaborative note-taking, sharing a PostgreSQL database.

📄 Compose file: [`stacks/paperless-affine/docker-compose.yml`](stacks/paperless-affine/docker-compose.yml)

**Volume mounts to update:**

| Path | Description |
|------|-------------|
| `/media/paperless/data` | Paperless application data |
| `/media/paperless/media` | Scanned documents |
| `/media/paperless/consume` | Drop folder (auto-import) |
| `/media/paperless/export` | Export directory |

**Services:**
- `paperless` — Document management with OCR (port 8222)
- `redis` — Task queue for Paperless
- `db` (pgvector/postgres) — Shared database for Paperless + AFFiNE
- `affine` — Notion-like collaborative workspace (port 3010)
- `affine_migration` — One-shot DB schema migration
- `affine_redis` — Dedicated Redis for AFFiNE

> **Credentials:** The default admin credentials in the compose file are for initial setup only. Change `PAPERLESS_ADMIN_PASSWORD` immediately after first login.

Access Paperless at `http://<host>:8222` | AFFiNE at `http://<host>:3010`

---

### RoMM (ROM Manager)

Self-hosted ROM management platform with metadata scraping from multiple sources.

📄 Compose file: [`stacks/romm/docker-compose.yml`](stacks/romm/docker-compose.yml)

**Environment variables to configure:**

```env
DB_PASSWD=<your-secure-password>
MARIADB_ROOT_PASSWORD=<your-secure-password>
MARIADB_PASSWORD=<your-secure-password>
ROMM_AUTH_SECRET_KEY=<generate with: openssl rand -hex 32>

# Metadata providers (optional but recommended)
SCREENSCRAPER_USER=<your-screenscraper-username>
SCREENSCRAPER_PASSWORD=<your-screenscraper-password>
RETROACHIEVEMENTS_API_KEY=<your-ra-api-key>
STEAMGRIDDB_API_KEY=<your-steamgriddb-api-key>
```

**Volume mounts to update:**

| Path | Description |
|------|-------------|
| `/path/to/library` | Your ROM library root |
| `/media/save` | Save states and save files |
| `/path/to/config` | RoMM config.yml (optional) |

Access at `http://<host>:8090`

---

### Nginx Proxy Manager

Reverse proxy with a web UI for managing domains, SSL certificates (Let's Encrypt), and routing.

Deploy via Portainer using the official compose from [nginxproxymanager.com](https://nginxproxymanager.com/setup/).

```yaml
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: nginx-proxy-manager-app-1
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'  # Admin UI
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt

volumes:
  npm_data:
  npm_letsencrypt:
```

Default admin login: `admin@example.com` / `changeme` — **change immediately after first login.**

---

## Standalone Containers

These run directly via `docker run` or simple single-service compose files, managed within the Portainer GUI.

| Container | Image | Port | Description |
|-----------|-------|------|-------------|
| `pihole` | `pihole/pihole` | 53, 80 | DNS-level ad blocker |
| `homeassistant` | `ghcr.io/home-assistant/home-assistant` | 8123 | Home automation |
| `homepage` | `ghcr.io/gethomepage/homepage` | 3000 | Dashboard / start page |
| `glance` | `glanceapp/glance` | 8080 | Self-hosted feed dashboard |
| `grocy` | `linuxserver/grocy` | 9283 | Grocery & household management |
| `vaultwarden` | `vaultwarden/server` | 8080 | Bitwarden-compatible password manager |
| `tailscale` | `tailscale/tailscale` | — | Zero-config VPN mesh |
| `cloudflared` | `cloudflare/cloudflared` | — | Cloudflare Tunnel (expose services securely) |

---

## Networking

### Internal Access

All services are on the same Docker bridge network within the Portainer LXC. Services in different stacks that need to communicate (e.g. paperless sharing postgres with affine) are placed in the same compose file or connected via a shared external Docker network.

### External Access via Cloudflare Tunnel

Cloudflared creates a secure outbound-only tunnel to Cloudflare's edge — no port forwarding needed on your router.

```bash
# Authenticate
docker run --rm -it cloudflare/cloudflared:latest tunnel login

# Create tunnel
docker run --rm cloudflare/cloudflared:latest tunnel create home-server
```

Then configure routes in the Cloudflare Zero Trust dashboard, pointing each hostname to your internal service (e.g. `paperless.yourdomain.com` → `http://192.168.1.20:8222`).

### Tailscale (VPN Access)

Tailscale provides secure remote access to your server from anywhere without exposing ports publicly.

```bash
docker run -d \
  --name tailscale \
  --hostname homeserver \
  --cap-add NET_ADMIN \
  --cap-add SYS_MODULE \
  -v /dev/net/tun:/dev/net/tun \
  -v tailscale_state:/var/lib \
  -e TS_AUTHKEY=<your-auth-key> \
  --restart unless-stopped \
  tailscale/tailscale:latest
```

### Pi-hole (DNS)

Set your router's DNS server to the Pi-hole container's IP to block ads network-wide. In Pi-hole's upstream DNS, point to your preferred resolver (e.g. Cloudflare `1.1.1.1` or your own Unbound instance).

---

## Storage Layout

```
/
├── media/
│   ├── music/              # Music library (Navidrome + Filebrowser)
│   ├── paperless/
│   │   ├── data/
│   │   ├── media/
│   │   ├── consume/        # Drop PDFs here for auto-import
│   │   ├── export/
│   │   ├── redis/
│   │   └── db/             # PostgreSQL data
│   └── save/               # RoMM save states
│
└── mnt/
    └── storage/
        └── appdata/
            ├── navidrome/
            ├── filebrowser/
            └── ...         # Per-service config directories
```

> **Tip:** If you're using a secondary drive for media, mount it at `/media` and use bind mounts in your compose files pointing to that path. This keeps OS/app data on the fast SSD and media on the larger HDD.

---

## Contributing

If you adapt this setup and improve on it, feel free to open a PR. Issues and suggestions welcome!

## License

MIT
