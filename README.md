# 🏠 Home Server Setup Guide

A complete guide to my self-hosted infrastructure running on Proxmox with Docker/LXC containers. This repo documents the full stack from bare metal to running services so you can replicate or adapt it for your own setup.

---

## 📋 Table of Contents

- [Design Goals](#design-goals)
- [Architecture Overview](#architecture-overview)
- [Hardware & OS](#hardware--os)
- [Proxmox Setup](#proxmox-setup)
- [LXC Containers](#lxc-containers)
  - [How LXCs Work](#how-lxcs-work)
  - [Portainer LXC (Docker Host)](#portainer-lxc-docker-host)
  - [Services LXC (Web/ML/Dev)](#services-lxc-webmldev)
- [Docker Stacks](#docker-stacks)
  - [Immich (Photo Management)](#immich-photo-management)
  - [Karakeep (Bookmark Manager)](#karakeep-bookmark-manager)
  - [Navidrome + Filebrowser (Music)](#navidrome--filebrowser-music)
  - [Paperless-NGX (Documents)](#paperless-ngx-documents)
  - [Obsidian LiveSync + CouchDB (Notes)](#obsidian-livesync--couchdb-notes)
  - [RoMM (ROM Manager)](#romm-rom-manager)
  - [Nginx Proxy Manager](#nginx-proxy-manager)
- [Standalone Containers](#standalone-containers)
- [Networking](#networking)
  - [Internal Access](#internal-access)
  - [External Access via Cloudflare Tunnel](#external-access-via-cloudflare-tunnel)
  - [NetBird (VPN Access)](#netbird-vpn-access)
  - [Pi-hole (DNS)](#pi-hole-dns)
  - [Pi-hole + NetBird (Ad Blocking Anywhere)](#pi-hole--netbird-ad-blocking-anywhere)
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
| `paperlessngx` | paperless, redis, postgres |
| `obsidian-livesync-couchdb` | couchdb |
| `romm` | romm, romm-db (mariadb) |
| `nginx-proxy-manager` | nginx-proxy-manager |

**Standalone containers:** pihole, homeassistant, homepage, glance, grocy, vaultwarden, netbird, cloudflared, portainer

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

### How LXCs Work

In Proxmox, an LXC container is lighter than a full virtual machine. A VM gets its own virtual hardware and kernel. An LXC shares the Proxmox host's Linux kernel, but still gives you an isolated userspace with its own filesystem, packages, services, IP address, and resource limits.

That makes LXCs a nice middle ground for homelab services:

- They boot fast and use less RAM than full VMs.
- They are easy to back up and restore from the Proxmox UI.
- They can get their own static IPs, bind mounts, CPU limits, and memory limits.
- They work well for infrastructure services that do not need a totally separate kernel.

There are tradeoffs. Because an LXC shares the host kernel, it is not the same isolation boundary as a VM. If you are running untrusted workloads, unusual kernel modules, or anything that needs its own kernel, use a VM instead.

For this setup, I use Alpine Linux LXCs because Alpine is small, fast, and simple. It uses `apk` for packages and OpenRC for services, so commands look like `apk add nginx` and `rc-update add nginx boot`.

Ubuntu LXCs are also a good choice, especially if you are newer to Linux or following guides that assume `apt` and `systemd`. Ubuntu usually has more familiar docs and fewer surprises with software that expects glibc or systemd. Alpine is leaner; Ubuntu is more familiar. Either works, but do not blindly copy Alpine commands into Ubuntu or Ubuntu commands into Alpine.

For Docker inside an LXC, enable nesting. Without it, Docker may fail because it needs container features inside the container:

```text
features: keyctl=1,nesting=1
```

If Docker-in-LXC gives you constant permission or filesystem issues, move that workload into a small VM. The guide uses an Alpine LXC for the Docker host because it is lightweight and works well for this setup, but a VM is the more conservative option.

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

- Keep Filebrowser private behind NetBird or your LAN if possible. It has write access to the music library.
- Use Navidrome for playback, playlists, users, and remote listening; use Filebrowser for uploads, folder cleanup, and quick file operations.
- Tag high-res files before or after upload with a tool like MusicBrainz Picard or beets. Navidrome relies heavily on embedded metadata, not just folder names.
- If you expose only one service remotely, expose Navidrome. Filebrowser is more of an admin tool.

---

### Paperless-NGX (Documents)

Paperless-NGX handles document intake, OCR, tagging, search, and long-term document storage. I split it out from AFFiNE so documents and notes are no longer coupled to the same database stack.

📄 Compose file: [`paperlessngx/docker-compose.yaml`](paperlessngx/docker-compose.yaml)

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
- `db` (postgres) — Paperless database

> **Credentials:** The default admin credentials in the compose file are for initial setup only. Change `PAPERLESS_ADMIN_PASSWORD` immediately after first login.

Access Paperless at `http://<host>:8222`

---

### Obsidian LiveSync + CouchDB (Notes)

AFFiNE has been replaced with Obsidian for notes. Obsidian keeps the actual notes as local Markdown files, while Self-hosted LiveSync uses CouchDB as the sync backend across devices. This fits the same philosophy as the rest of the homelab: keep the important data portable, then layer self-hosted services on top.

📄 Compose file: [`obsidian-livesync-couchdb/docker-compose.yaml`](obsidian-livesync-couchdb/docker-compose.yaml)

**Environment variables needed:**

```env
COUCHDB_USER=<your-couchdb-admin-user>
COUCHDB_PASSWORD=<your-secure-couchdb-password>
```

**Volume mounts to update before deploying:**

| Path | Description |
|------|-------------|
| `/mnt/storage/appdata/couchdb/data` | CouchDB database files |
| `/mnt/storage/appdata/couchdb/etc` | CouchDB local configuration |

**Services:**
- `couchdb` — Sync backend for Obsidian Self-hosted LiveSync (port 5984)

Before first boot, create the host folders and give CouchDB ownership:

```bash
mkdir -p /mnt/storage/appdata/couchdb/data /mnt/storage/appdata/couchdb/etc
chown -R 5984:5984 /mnt/storage/appdata/couchdb
```

After deploying, initialize CouchDB for Self-hosted LiveSync using the official init script:

```bash
curl -s https://raw.githubusercontent.com/vrtmrz/obsidian-livesync/main/utils/couchdb/couchdb-init.sh \
  | hostname=http://<host>:5984 username=<COUCHDB_USER> password=<COUCHDB_PASSWORD> bash
```

Then install these Obsidian plugins:

- Self-hosted LiveSync — Syncs the vault through CouchDB
- Excalidraw — Drawings and visual notes inside Obsidian
- Any other community plugins you use for your workflow

In Obsidian, configure Self-hosted LiveSync with the CouchDB endpoint, database name, username, password, and an end-to-end encryption passphrase. Use the plugin setup wizard or setup URI flow when adding more devices.

Keep CouchDB private behind NetBird or a trusted reverse proxy with HTTPS. Do not expose it directly to the public internet without auth, TLS, and careful access controls.

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
| `netbird` | `netbirdio/netbird` | — | Open-source Zero-trust VPN mesh |
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

### NetBird (VPN Access)

[NetBird](https://github.com/netbirdio/netbird) is an open-source overlay network tool that establishes secure peer-to-peer connections between machines using WireGuard, without complex firewall configuration.

#### Benefits of NetBird:
- **100% Open Source:** NetBird is fully open-source (under BSD 3-Clause license) and can be entirely self-hosted (dashboard, management server, signal server, and turn relay).
- **Zero-Trust Network:** Control who can access what using central Access Control Policies (ACLs).
- **Mullvad VPN Support (Coexistence):** While NetBird does not have a one-click paid Mullvad integration like Tailscale, its open-source routing engine allows it to easily coexist with privacy VPNs like Mullvad. You can route specific public internet traffic through Mullvad exit nodes using community scripts (e.g., [netbird-mullvad-bypass](https://github.com/mischw/netbird-mullvad-bypass)) or custom routing rules, retaining access to private NetBird nodes while anonymizing external traffic.
- **Identity Provider (IdP) Integration:** Connect with Okta, Keycloak, Google Workspace, Azure AD, etc., for single sign-on (SSO).
- **Centralized Subnet Routing:** Easily share subnets or entire LAN networks using specific routing peers with built-in route health checks.

For this layout, the cleanest setup is to run NetBird inside the Portainer LXC or another small "network services" LXC, then access published Docker ports over the NetBird IP.

#### Installing NetBird on the LXC
Install the NetBird client directly on the Proxmox LXC and connect it to your network:

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
netbird up --setup-key <YOUR_SETUP_KEY>
```

#### Docker Client Setup
If you prefer running the NetBird client inside a Docker container, use the following stack definition. Ensure you persist the client state directory so restarts do not register duplicate peers in your NetBird dashboard:

```yaml
services:
  netbird:
    image: netbirdio/netbird:latest
    container_name: netbird
    hostname: homeserver
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_ADMIN
      - SYS_RESOURCE
    environment:
      - NB_SETUP_KEY=${NB_SETUP_KEY:?Set NB_SETUP_KEY}
    volumes:
      - netbird_state:/var/lib/netbird
    devices:
      - /dev/net/tun:/dev/net/tun
    network_mode: host
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1

volumes:
  netbird_state:
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

### Pi-hole + NetBird (Ad Blocking Anywhere)

Pi-hole handles DNS filtering. NetBird makes that DNS server reachable from your phone, laptop, or tablet even when you are away from home. Together they act like a universal ad blocker for any device connected to your NetBird network.

Recommended setup:

1. Run Pi-hole on the Docker/LXC host and publish port `53` on the host.
2. Run NetBird client on that same host/LXC.
3. Find the host's NetBird IP by checking the client status:

```bash
netbird status --detail
```

4. In the NetBird admin console, go to **DNS** -> **Nameservers** -> **Add Nameserver**.
5. Choose **Custom DNS**, enter a name (e.g., `Pi-hole`), and enter the host's NetBird IP (e.g., `10.x.y.z`).
6. Leave the **Match Domains** field empty (so it resolves all domains through Pi-hole) and assign the configuration to the relevant peer group (e.g., `All`).
7. Save the nameserver.

Test from a remote device:

```bash
nslookup doubleclick.net 10.x.y.z
```

Then open Pi-hole's query log. You should see the remote NetBird device making DNS requests.

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
- `/mnt/storage/appdata/couchdb`
- Portainer data
- Pi-hole `/etc/pihole`
- Nginx Proxy Manager `/data` and `/etc/letsencrypt`
- Obsidian vaults on each client device

Database-backed apps should also get logical dumps:

```bash
# PostgreSQL example
docker exec -t paperless-postgres pg_dump -U paperless paperless > paperless-postgres-backup.sql

# MariaDB example
docker exec -t romm-db mariadb-dump -u root -p romm > romm-backup.sql
```

For CouchDB, prefer replication to another CouchDB target or stop the container before taking a filesystem-level backup of `/mnt/storage/appdata/couchdb`.

### Updates

Do updates in small batches:

```bash
docker compose pull
docker compose up -d
docker image prune
```

For critical services like Immich, Paperless, CouchDB/Obsidian LiveSync, and RoMM, read release notes before major upgrades. Databases deserve extra caution: take a backup before changing image tags or versions.

---

## Security Checklist

- Do not expose Proxmox, Portainer, Pi-hole, Home Assistant, or database ports directly to the internet.
- Use NetBird for admin access and Cloudflare Tunnel only for services meant to be reachable by a public hostname.
- Treat Filebrowser like an admin surface because it can upload, rename, and delete files in mounted folders.
- Treat CouchDB like a database, not a public notes app. Keep it behind NetBird or a locked-down HTTPS reverse proxy.
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

### NetBird Works but Pi-hole Does Not Block Remotely

- Confirm the remote device is connected to NetBird.
- Confirm NetBird DNS nameserver settings have Pi-hole set as a custom nameserver.
- Confirm the custom nameserver is active and assigned to the correct peer group.
- Confirm the nameserver IP is the NetBird IP of the host running Pi-hole.
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

### Obsidian LiveSync Cannot Reach CouchDB

- Confirm CouchDB is reachable at `http://<host>:5984/_utils`.
- Confirm the official Self-hosted LiveSync CouchDB init script completed successfully.
- Confirm the Obsidian plugin is using the same database name, username, password, and endpoint.
- If syncing mobile devices, use HTTPS through NetBird, Cloudflare Tunnel, or a reverse proxy with a valid certificate.
- Check CouchDB logs with `docker logs obsidian-livesync-couchdb` for auth, CORS, or permission errors.

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
            ├── couchdb/
            │   ├── data/          # Obsidian LiveSync CouchDB data
            │   └── etc/           # CouchDB local config
            └── ...         # Per-service config directories
```

> **Tip:** If you're using a secondary drive for media, mount it at `/media` and use bind mounts in your compose files pointing to that path. This keeps OS/app data on the fast SSD and media on the larger HDD.

---

## Contributing

If you adapt this setup and improve on it, feel free to open a PR. Issues and suggestions welcome!

## License

MIT
