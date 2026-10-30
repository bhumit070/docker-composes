# Homelab

Docker Compose stack in `compose.yaml`; all services share the `homelab_network` network.
Bring everything up with the user's alias `dcu` (`docker compose --env-file .env up -d`).

## nginx reverse proxy

- Each app gets `https://<name>.bhumit070.bid` via a file in `nginx/conf.d/`, using the
  certbot wildcard cert configured once in `nginx/nginx.conf`.
- Upstreams are addressed by **container name**, but always through a variable:

  ```nginx
  location / {
      set $upstream http://<container>:<port>; # variable => re-resolved via resolver in nginx.conf
      proxy_pass $upstream;
      include /etc/nginx/conf.d/proxy_params.inc;
  }
  ```

  A literal `proxy_pass http://<container>:<port>;` makes nginx resolve the name to an IP only
  once, at startup/reload, and cache it. When the container is recreated it gets a new IP and
  nginx returns 502 until reloaded. The variable form uses the Docker DNS `resolver 127.0.0.11`
  in `nginx.conf`, so new IPs are picked up automatically. Keep this pattern for new sites.
- Services with a domain publish **no host ports**: reach them only via nginx (HTTPS).
  Only nginx (80/443) and jellyfin's 7359/udp client discovery are published.
- New `conf.d` files are only loaded on reload: `docker exec nginx nginx -s reload`
  (until then the new subdomain fails TLS with "unrecognized name").

## Storage

- `/media/bhumit070/Data` is NTFS (`fuseblk`): no chown/chmod. Postgres data dirs must live
  off it, under `~/docker/<name>` (immich, vikunja, penpot already do).

## Secrets

- `.env` holds secrets and is git-ignored; never commit it. The GitHub repo is public.

## Penpot (design.bhumit070.bid)

- Flags (frontend + backend): `disable-email-verification enable-access-tokens enable-mcp`.
- `penpot-mcp` is Penpot's MCP server for Claude; penpot-frontend proxies it at
  `/mcp/stream` (HTTP) and `/mcp/ws` (websocket). Enable it per user in
  Your account → Integrations.
