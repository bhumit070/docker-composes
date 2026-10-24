# Homelab

Welcome to my personal Homelab! This repository contains the Docker Compose configuration to deploy a comprehensive suite of self-hosted applications and services.

## Overview

This setup uses Docker Compose to orchestrate various services, ranging from media servers and photo management to homelab dashboards and container management. 

### Included Services

Here is a list of services deployed in this homelab:

| Service | Description | URL | Exposed Port(s) |
| :--- | :--- | :--- | :--- |
| **[Immich](https://immich.app/)** | High-performance self-hosted photo and video backup solution. | https://photos.bhumit070.bid | `2283` |
| **[Jellyfin](https://jellyfin.org/)** | Free Software Media System for streaming media. | https://media.bhumit070.bid | `8096` (TCP), `7359` (UDP) |
| **[Dozzle](https://dozzle.dev/)** | Real-time log viewer for Docker containers. | https://dozzle.bhumit070.bid | `5050` |
| **[Glance](https://github.com/glanceapp/glance)** | A self-hosted dashboard for your homelab. | https://dashboard.bhumit070.bid | Internal |
| **[Filebrowser](https://filebrowser.org/)** | Web-based file manager. | https://files.bhumit070.bid | `8080` |
| **[Vikunja](https://vikunja.io/)** | Open-source, self-hosted to-do app. | https://tasks.bhumit070.bid | `3456` |
| **[Portainer](https://www.portainer.io/)** | Universal container management platform. | https://portainer.bhumit070.bid | `9443` |
| **[Nginx](https://nginx.org/)** | High-performance web server and reverse proxy. | - | `80`, `443` |
| **[Certbot](https://certbot.eff.org/)** | Issues and renews the `*.bhumit070.bid` Let's Encrypt certificate via Cloudflare DNS. | - | Internal |
| **[Navidrome](https://www.navidrome.org/)** | Modern Music Server and Streamer compatible with Subsonic/Airsonic. | - | `4533` |
| **[Jellystat](https://github.com/CyferShepard/Jellystat)** | Statistics for Jellyfin. | https://jellystat.bhumit070.bid | `3000` |
| **[Kavita](https://www.kavitareader.com/)** | Fast, feature-rich, cross-platform reading server (Manga/Comics/Books). | https://books.bhumit070.bid | `5000` |

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

To add an app, copy one of the files in `nginx/conf.d/`, change `server_name` and `proxy_pass`, then run `docker compose restart nginx`. The certificate already covers every subdomain.

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

All services communicate over a custom Docker bridge network called `homelab_network` for secure internal communication. Some services (like Glance) do not expose public ports and are intended to be routed through a reverse proxy (like Nginx).
