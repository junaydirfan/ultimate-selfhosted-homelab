# 🏠 Home Server Setup Guide

A complete guide to my self-hosted infrastructure running on Proxmox with Docker/LXC containers. This repo documents the full stack from bare metal to running services so you can replicate or adapt it for your own setup.

---

## Design Goals

- Keep admin access private by default with Tailscale instead of exposing dashboards directly to the internet.
- Use Cloudflare Tunnel only for services that genuinely need public hostnames.
- Store app data and media outside containers so stacks can be rebuilt without losing state.
- Keep source-of-truth data in plain folders, then layer purpose-built apps on top for browsing, streaming, search, or automation.
- Keep secrets in Portainer environment variables or ignored `.env` files, never committed compose files.
- Prefer boring, easy-to-restore infrastructure over clever one-off tweaks.

---

## 📋 Table of Contents

- [Design Goals](#design-goals)
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
  - [Internal Access](#internal-access)
  - [External Access via Cloudflare Tunnel](#external-access-via-cloudflare-tunnel)
  - [Tailscale (VPN Access)](#tailscale-vpn-access)
  - [Pi-hole (DNS)](#pi-hole-dns)
  - [Pi-hole + Tailscale (Ad Blocking Anywhere)](#pi-hole--tailscale-ad-blocking-anywhere)
- [Backups & Maintenance](#backups--maintenance)
- [Security Checklist](#security-checklist)
- [Troubleshooting](#troubleshooting)
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

**Suggested `.env` workflow:**

```bash
# Create strong secrets for stacks that need them
openssl rand -base64 32
openssl rand -hex 32

# Keep real values out of git
cp .env.example .env
chmod 600 .env
```

Use a separate `.env.example` beside each compose file with placeholder values only. That makes the repo reusable without leaking private keys, passwords, API tokens, or real domain names.

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

📄 Compose file: [`karakeep/docker-compose.yaml`](karakeep/docker-compose.yaml)

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

Navidrome is a self-hosted music server (Subsonic-compatible). Filebrowser provides a web UI for managing the same music directory.

This pairing is intentional: Filebrowser is the simple upload/admin surface, and Navidrome is the polished playback/indexing layer. I use Filebrowser to drag and drop high-res audio files into `/media/music`, then Navidrome scans that folder and makes the library available from anywhere through the web UI or a Subsonic-compatible app. The files stay as normal folders on disk, so the library is not trapped inside either app.

📄 Compose file: [`navidrome-filebrowser/docker-compose.yaml`](navidrome-filebrowser/docker-compose.yaml)

**Volume mounts to update before deploying:**

| Path | Description |
|------|-------------|
| `/media/music` | Your music library directory |
| `/mnt/storage/appdata/navidrome` | Navidrome database/config |
| `/mnt/storage/appdata/filebrowser` | Filebrowser config |

**Services:**
- `navidrome` — Music streaming server (port 4533)
- `filebrowser` — Web file manager for music library (port 8088)

**Workflow:**

1. Upload albums, singles, or cleaned-up folder trees through Filebrowser at `http://<host>:8088`.
2. Filebrowser writes directly into `/media/music`, which is mounted into Navidrome as `/music`.
3. Navidrome rescans on the schedule set by `ND_SCANSCHEDULE=1m` in the compose file.
4. Stream from Navidrome at `http://<host>:4533` or connect a Subsonic-compatible app such as Symfonium, Finamp, or DSub.

**Suggested music layout:**

```text
/media/music/
├── Artist/
│   └── 2024 - Album Name/
│       ├── 01 - Track Name.flac
│       ├── 02 - Track Name.flac
│       └── cover.jpg
└── Compilations/
    └── 2023 - Compilation Name/
```

**Notes:**

- Keep Filebrowser private behind Tailscale or your LAN if possible. It has write access to the music library.
- Use Navidrome for playback, playlists, users, and remote listening; use Filebrowser for uploads, folder cleanup, and quick file operations.
- Tag high-res files before or after upload with a tool like MusicBrainz Picard or beets. Navidrome relies heavily on embedded metadata, not just folder names.
- If you expose only one service remotely, expose Navidrome. Filebrowser is more of an admin tool.

---

### Paperless-NGX + AFFiNE (Documents & Notes)

Combined stack for document management and collaborative note-taking, sharing one PostgreSQL instance.

📄 Compose file: [`paperlessngx-AFFiNE/docker-compose.yaml`](paperlessngx-AFFiNE/docker-compose.yaml)

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

> **Database note:** If AFFiNE uses its own `AFFINE_DB_USER`, `AFFINE_DB_PASS`, and `affine` database on the shared Postgres container, create that database/user before starting the AFFiNE service or add an init script. Sharing the Postgres instance is fine; sharing the same application database is not recommended.

Access Paperless at `http://<host>:8222` | AFFiNE at `http://<host>:3010`

---

### RoMM (ROM Manager)

Self-hosted ROM management platform with metadata scraping from multiple sources.

📄 Compose file: [`romm/docker-compose.yaml`](romm/docker-compose.yaml)

**Environment variables to configure:**

```env
ROMM_DB_PASS=<your-secure-password>
ROMM_DB_ROOT_PASS=<your-secure-password>
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

Most services publish ports on the Portainer LXC and are accessed by LAN IP, for example `http://192.168.1.20:8222`.

Services in the same compose file can talk to each other by service name, such as `db:5432` or `redis:6379`. Services in different stacks should only share a Docker network when they genuinely need to communicate. Keep databases private to their stack unless you are intentionally sharing one database server.

Suggested pattern:

```bash
# Optional shared network for reverse-proxied apps
docker network create proxy
```

Then attach only the apps that Nginx Proxy Manager needs to reach.

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

Tailscale provides secure remote access to your server from anywhere without exposing admin ports publicly. For this layout, the cleanest setup is to run Tailscale inside the Portainer LXC or another small "network services" LXC, then access published Docker ports over the Tailscale IP.

Install Tailscale on the LXC and prevent the DNS server itself from accepting tailnet DNS settings:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --accept-dns=false
```

In the Tailscale admin console, consider disabling key expiry for trusted always-on server nodes so remote access does not randomly break. Only do this for devices you physically control.

If you prefer the Docker image, persist Tailscale state so restarts do not create a new machine every time:

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: homeserver
    restart: unless-stopped
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY:?Set TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_ACCEPT_DNS=false
    volumes:
      - tailscale_state:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW

volumes:
  tailscale_state:
```

### Pi-hole (DNS)

Set your router's DHCP DNS server to the Pi-hole host/LXC IP to block ads and trackers network-wide at home. In Pi-hole's upstream DNS, point to your preferred resolver such as Cloudflare `1.1.1.1`, Quad9 `9.9.9.9`, or your own Unbound instance.

Minimal Docker Compose example:

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    hostname: pihole
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8081:80/tcp"
    environment:
      TZ: America/Toronto
      FTLCONF_webserver_api_password: ${PIHOLE_WEB_PASSWORD:?Set PIHOLE_WEB_PASSWORD}
      FTLCONF_dns_listeningMode: all
      FTLCONF_dns_upstreams: "1.1.1.1;9.9.9.9"
    volumes:
      - pihole_etc:/etc/pihole

volumes:
  pihole_etc:
```

Access the admin UI at `http://<host>:8081/admin`.

### Pi-hole + Tailscale (Ad Blocking Anywhere)

Pi-hole handles DNS filtering. Tailscale makes that DNS server reachable from your phone, laptop, or tablet even when you are away from home. Together they act like a universal ad blocker for any device connected to your tailnet.

Recommended setup:

1. Run Pi-hole on the Docker/LXC host and publish port `53` on the host.
2. Run Tailscale on that same host/LXC with `--accept-dns=false`.
3. Find the host's Tailscale IP:

```bash
tailscale ip -4
```

4. In the Tailscale admin console, go to **DNS**.
5. Add a **Custom nameserver** using the Pi-hole host's Tailscale IP, for example `100.x.y.z`.
6. Enable **Override DNS servers** so tailnet devices use Pi-hole instead of whatever DNS the current Wi-Fi or mobile network provides.
7. Keep MagicDNS enabled so tailnet device names still resolve.

If you use a Tailscale exit node, edit the Pi-hole nameserver in the DNS page and enable **Use with exit node**.

Test from a remote device:

```bash
nslookup doubleclick.net 100.x.y.z
```

Then open Pi-hole's query log. You should see the remote tailnet device making DNS requests.

**Recommended blocklists:**

Start with one good list, then allowlist only what breaks. Too many overlapping lists make troubleshooting painful.

| List | Use case | URL |
|------|----------|-----|
| HaGeZi Multi PRO | Balanced privacy, ads, tracking, malware | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt` |
| HaGeZi Multi PRO++ | Stricter blocking, higher chance of breakage | `https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.plus.txt` |

Full repo: [hagezi/dns-blocklists](https://github.com/hagezi/dns-blocklists)

After adding or changing adlists:

```bash
docker exec -it pihole pihole -g
```

Limits to know:

- DNS blocking will not remove every ad, especially YouTube, Twitch, Instagram, TikTok, and some in-app ads.
- Some apps hardcode DNS or use encrypted DNS. You may need firewall rules if you want to force all LAN DNS through Pi-hole.
- If a site breaks, check the Pi-hole query log and allowlist the specific blocked domain instead of disabling the whole list.

---

## Backups & Maintenance

Backups are the difference between "fun homelab" and "weekend gone." Test restores before you trust any setup.

### Proxmox Backups

Use Proxmox scheduled backups for LXCs/VMs:

- Back up the Portainer LXC and Services LXC on a schedule.
- Store backups on a different disk, NAS, or Proxmox Backup Server.
- Keep at least one offline or offsite copy for important data.
- Test restoring an LXC to a new CTID before you need it.

### Docker Data Backups

Back up bind mounts and named volumes, especially:

- `/media/paperless`
- `/media/music`
- `/media/save`
- `/mnt/storage/appdata`
- Portainer data
- Pi-hole `/etc/pihole`
- Nginx Proxy Manager `/data` and `/etc/letsencrypt`

Database-backed apps should also get logical dumps:

```bash
# PostgreSQL example
docker exec -t postgres pg_dumpall -U paperless > postgres-backup.sql

# MariaDB example
docker exec -t romm-db mariadb-dump -u root -p romm > romm-backup.sql
```

### Updates

Do updates in small batches:

```bash
docker compose pull
docker compose up -d
docker image prune
```

For critical services like Immich, Paperless, AFFiNE, and RoMM, read release notes before major upgrades. Databases deserve extra caution: take a backup before changing image tags or versions.

---

## Security Checklist

- Do not expose Proxmox, Portainer, Pi-hole, Home Assistant, or database ports directly to the internet.
- Use Tailscale for admin access and Cloudflare Tunnel only for services meant to be reachable by a public hostname.
- Treat Filebrowser like an admin surface because it can upload, rename, and delete files in mounted folders.
- Change default credentials immediately after first login.
- Enable 2FA/MFA wherever the app supports it.
- Use strong unique passwords and store them in Vaultwarden or another password manager.
- Keep `.env` files, API keys, auth tokens, and tunnel credentials out of git.
- Prefer pinned major versions for databases instead of `latest`.
- Keep Proxmox, LXCs, Docker images, and application stacks updated.
- Review Cloudflare Tunnel public hostnames regularly and remove anything you no longer need.

---

## Troubleshooting

### Docker in LXC Does Not Start

Check that nesting is enabled in the Proxmox CT config:

```text
features: keyctl=1,nesting=1
```

Restart the container after changing this.

### Tailscale Works but Pi-hole Does Not Block Remotely

- Confirm the remote device is connected to Tailscale.
- Confirm Tailscale DNS has Pi-hole set as a custom nameserver.
- Confirm **Override DNS servers** is enabled.
- Confirm the nameserver IP is the Tailscale IP of the host running Pi-hole.
- Check Pi-hole query log while loading a site from the remote device.

### A Service Cannot Reach Its Database

- Use the compose service name as the hostname, not `localhost`.
- Confirm both services are in the same compose file or Docker network.
- Check healthchecks with `docker ps`.
- Check logs with `docker logs <container-name>`.

### File Permissions Break Media or Imports

For bind mounts like `/media/music` or `/media/paperless`, make sure the container user can read and write the host folder. LinuxServer.io containers usually use `PUID` and `PGID`; other images may run as root or a fixed internal user.

### Navidrome Does Not Show New Music

- Confirm Filebrowser uploaded the files into `/media/music`, not a nested path like `/media/music/music`.
- Check that Navidrome can read the files inside the container with `docker exec -it navidrome ls /music`.
- Wait for the next scan, or restart Navidrome if you want to force a quick re-index.
- Check metadata with a tag editor if albums appear under the wrong artist or as unknown.
- Check logs with `docker logs navidrome` for permission errors or unsupported file formats.

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
