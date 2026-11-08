# Homelab

Welcome to my personal Homelab! This repository contains the Docker Compose configuration to deploy a comprehensive suite of self-hosted applications and services.

## Overview

This setup uses Docker Compose to orchestrate various services, ranging from media servers and photo management to homelab dashboards and container management. 

### Included Services

Here is a list of services deployed in this homelab:

| Service | Description | URL | Exposed Port(s) |
| :--- | :--- | :--- | :--- |
| **[Immich](https://immich.app/)** | High-performance self-hosted photo and video backup solution. | https://photos.bhumit070.bid | Internal |
| **[Jellyfin](https://jellyfin.org/)** | Free Software Media System for streaming media. | https://media.bhumit070.bid | `7359` (UDP, client discovery) |
| **[Dozzle](https://dozzle.dev/)** | Real-time log viewer for Docker containers. | https://dozzle.bhumit070.bid | Internal |
| **[Glance](https://github.com/glanceapp/glance)** | A self-hosted dashboard for your homelab. | https://dashboard.bhumit070.bid | Internal |
| **[Filebrowser](https://filebrowser.org/)** | Web-based file manager. | https://files.bhumit070.bid | Internal |
| **[Vikunja](https://vikunja.io/)** | Open-source, self-hosted to-do app. | https://tasks.bhumit070.bid | Internal |
| **[Portainer](https://www.portainer.io/)** | Universal container management platform. | https://portainer.bhumit070.bid | Internal |
| **[Nginx](https://nginx.org/)** | High-performance web server and reverse proxy. | - | `80`, `443` |
| **[Certbot](https://certbot.eff.org/)** | Issues and renews the `*.bhumit070.bid` Let's Encrypt certificate via Cloudflare DNS. | - | Internal |
| **[Navidrome](https://www.navidrome.org/)** | Modern Music Server and Streamer compatible with Subsonic/Airsonic. | - | `4533` |
| **[Jellystat](https://github.com/CyferShepard/Jellystat)** | Statistics for Jellyfin. | https://jellystat.bhumit070.bid | Internal |
| **[Kavita](https://www.kavitareader.com/)** | Fast, feature-rich, cross-platform reading server (Manga/Comics/Books). | https://books.bhumit070.bid | Internal |
| **[Penpot](https://penpot.app/)** | Open-source design and prototyping platform. | https://design.bhumit070.bid | Internal |
| **[Stirling PDF](https://www.stirlingpdf.com/)** | Self-hosted toolkit to merge, split, convert and edit PDFs. | https://pdf.bhumit070.bid | Internal |
| **[Seafile](https://www.seafile.com/)** | File sync and share with desktop/mobile clients. SeaDoc and AI features are disabled. | https://storage.bhumit070.bid | Internal |
| **[Send Any Webhook to Discord](https://github.com/bhumit070/send-anywebhook-to-discord)** | Receives webhooks from any service (e.g. Vikunja) and relays them to Discord using templates. Auth is disabled (tailnet-only). | https://webhooks.bhumit070.bid | Internal |

---

## Configuration

This project relies on environmental variables stored in a `.env` file for configuration. Copy the example below or create your own based on your environment.

### Environment Variables (`.env`)

You need to create a `.env` file in the root of the `homelab` directory:

```env
# IMMICH ENV
IMMICH_UPLOAD_LOCATION=./library
IMMICH_DB_DATA_LOCATION=~/docker/immich_postgres
TZ=Asia/Kolkata
IMMICH_VERSION=v2
IMMICH_DB_PASSWORD=postgres
IMMICH_DB_USERNAME=postgres
IMMICH_DB_DATABASE_NAME=immich

# JELLYFIN ENV
JELLYFIN_MEDIA_LOCATION="/media/bhumit070/Data/"

# VIKUNJA ENV
VIKUNJA_DB_PASSWORD="your_secure_password"

# NAVIDROME ENV
NAVIDROME_MUSIC_PATH="/media/bhumit070/Data/media/music"

# KAVITA ENV
KAVITA_BOOKS_COLLECTION_LOCATION="/media/bhumit070/Data/media/books/"

# PENPOT ENV
PENPOT_SECRET_KEY="generate_with_openssl_rand_hex_32"
PENPOT_DB_PASSWORD="your_secure_password"

# SEND ANY WEBHOOK ENV
WEBHOOKS_DB_PASSWORD="your_secure_password"
WEBHOOKS_APP_SECRET="generate_with_openssl_rand_hex_32"

# SEAFILE ENV
SEAFILE_DB_PASSWORD="your_secure_password"
SEAFILE_DB_ROOT_PASSWORD="your_secure_password"
SEAFILE_REDIS_PASSWORD="your_secure_password"
SEAFILE_JWT_PRIVATE_KEY="generate_with_openssl_rand_hex_32"
SEAFILE_ADMIN_EMAIL="you@example.com"     # first start only
SEAFILE_ADMIN_PASSWORD="your_secure_password"  # first start only
```

*Note: Change the paths like `/media/bhumit070/Data/...` and database passwords to match your system and desired security requirements.*

## Domains & HTTPS

Every app has its own subdomain of `bhumit070.bid`, served over HTTPS. A Cloudflare DNS record `*` (type A, **DNS only**, not proxied) points to the Tailscale IP of `codingmachine`, so the apps are only reachable from devices on the tailnet.

- Nginx routes each subdomain to its container via `nginx/conf.d/<app>.conf`. Shared proxy settings live in `nginx/conf.d/proxy_params.inc`.
- `nginx/conf.d/redirect.conf` redirects every `http://*.bhumit070.bid` request to HTTPS.
- `http://codingmachine` (the Tailscale machine name) still serves Glance over plain HTTP, since a public certificate can't cover it.

The `certbot` service gets a wildcard certificate for `*.bhumit070.bid` using a Cloudflare DNS challenge, so the server never has to be reachable from the internet. It checks for renewal every 12 hours, and nginx reloads every 6 hours to pick up renewed certificates. One-time setup:

1. In Cloudflare, create an API token (My Profile → API Tokens → *Edit zone DNS* template) scoped to the `bhumit070.bid` zone.
2. Save it on the homelab (this path is gitignored, so it is never committed):
   ```bash
   mkdir -p docker_data/certbot
   echo "dns_cloudflare_api_token = <your-token>" > docker_data/certbot/cloudflare.ini
   chmod 600 docker_data/certbot/cloudflare.ini
   ```
3. `docker compose up -d`. Nginx waits until the certificate exists.

To add an app:

1. Add the service to `compose.yaml` on `homelab_network`, without host `ports:` (apps are reached only through nginx).
2. Copy one of the files in `nginx/conf.d/` and change `server_name` and the `set $upstream http://<container>:<port>;` line. Keep the `$upstream` variable instead of a literal `proxy_pass` URL, so nginx re-resolves the container's IP after it is recreated.
3. Run `docker exec nginx nginx -s reload`. The certificate already covers every subdomain.
4. Add a bookmark in `glance/config/home.yml`, then `docker restart glance` (the repo is on NTFS, so Glance doesn't notice config changes on its own).

## Directories & Volumes Structure

Before deploying the stack, ensure the required host directories are available and permissions are properly set. The compose file relies on the following local directories for persistent storage:

- `./library` (Immich uploads)
- `~/docker/immich_postgres` (Immich database)
- `./jellyfin-config` (Jellyfin config)
- `./jellyfin-cache` (Jellyfin cache)
- `./glance/config/` (Glance config)
- `./glance/assets/` (Glance assets)
- `./docker_data/filebrowser_database` (Filebrowser database)
- `./docker_data/filebrowser_config` (Filebrowser config)
- `./docker_data/vikunja_files` (Vikunja file attachments)
- `~/docker/vikunja_db` (Vikunja database)
- `~/docker/penpot_postgres` (Penpot database)
- `~/docker/webhooks_postgres` (Send Any Webhook database)
- `./docker_data/seafile` (Seafile config, logs and file blocks)
- `~/docker/seafile_mysql` (Seafile MariaDB database)
- `./docker_data/stirling-pdf` (Stirling PDF data, config and logs)
- `./docker_data/portainer_data` (Portainer data)
- `./nginx/nginx.conf` (Nginx main config)
- `./nginx/conf.d` (Nginx supplementary configs)
- `./docker_data/navidrome` (Navidrome data)
- `./docker_data/kavita_config` (Kavita config)
- `./docker_data/letsencrypt` (TLS certificates)
- `./docker_data/certbot/cloudflare.ini` (Cloudflare API token)

## Getting Started

1. **Clone the repository** (or navigate to your `homelab` folder).
2. **Create the environment file**:
   ```bash
   cp .env-example .env
   # Or create one manually as shown above and configure your paths.
   ```
3. **Start the services** in detached mode:
   ```bash
   docker compose up -d
   ```
4. **Stop the services**:
   ```bash
   docker compose down
   ```
5. **Update containers** to the latest image versions:
   ```bash
   docker compose pull
   docker compose up -d
   ```

## Network

All services communicate over a custom Docker bridge network called `homelab_network` for secure internal communication. Apps don't publish host ports; they are reached only through Nginx over HTTPS. The exceptions are Nginx itself (`80`, `443`) and Jellyfin's `7359/udp` client discovery.
