# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS (optimized for low specs and minimal maintenance).
All services run behind Caddy (reverse proxy + automatic TLS).
One `.env` file drives the entire stack.

## Stack

| Service | Image | Role | Limit |
|---|---|---|---|
| Caddy | `caddy:latest` | Reverse proxy, TLS | 128M |
| AdGuard Home | `adguard/adguardhome:latest` | DNS + ad blocking | 192M |
| AIOMetadata | `ghcr.io/cedya77/aiometadata:latest` | Stremio catalog & metadata generator | 256M |
| AIOMetadata Cache | `redis:alpine` | Redis for AIOMetadata | 32M |
| AIOStreams | `ghcr.io/viren070/aiostreams:latest`| Stremio addon aggregator | 448M |
| Beszel | `henrygd/beszel:latest` | Monitoring dashboard | 64M |
| Beszel Agent | `henrygd/beszel-agent:latest` | Host metrics (`network_mode: host`) | 64M |
| Dispatcharr | `ghcr.io/dispatcharr/dispatcharr:latest` | IPTV stream management | 1024M |
| Filebrowser | `filebrowser/filebrowser:v2-alpine` | Web file manager (serves `/opt`) | 32M |
| Ghostfolio | `ghostfolio/ghostfolio:latest` | Portfolio tracker | 384M |
| Ghostfolio DB | `postgres:15-alpine` | Postgres for Ghostfolio | 64M |
| Ghostfolio Cache | `redis:alpine` | Redis for Ghostfolio | 32M |
| Honey | `ghcr.io/dani3l0/honey:latest` | Dashboard / start page | 48M |
| MediaFlow Proxy | `ghcr.io/mhdzumair/mediaflow-proxy-light:latest` | Debrid media proxy | 96M |
| Warp | `caomingjun/warp:latest` | Cloudflare HTTP proxy for MediaFlow | 64M |
| Watchtower | `nickfedor/watchtower:latest` | Auto image updates | 64M |
| Uptime Kuma | `louislam/uptime-kuma:latest` | Uptime monitoring & status page | 128M |
| Wallos | `bellamy/wallos:latest` | Subscription tracker | 64M |
| **Total** |  |  | **3200M** |

## Repo structure

```
compose.yaml          # Root: includes all apps via `include:`
.env.example          # Template — copy to .env and fill
.gitattributes        # Enforces LF line endings
.gitignore
CLAUDE.md
README.md
apps/
  <service>/
    compose.yaml
    Caddyfile         # caddy only
    config.json       # honey only
    redis.conf        # dispatcharr only
    uwsgi.ini         # dispatcharr only
```

Data is persisted under `$DOCKER_DATA_DIR` (default `/opt/docker/data`), never in the repo.

## Network

All services share `vps_network` (defined in root `compose.yaml`). Services talk to each other by container name.

Only Caddy and AdGuard expose host ports (`80`, `443`, `53`, `853`). All other services use `expose:` only — never `ports:`.

Beszel Agent uses `network_mode: host` and communicates with the hub via a Unix socket volume (`beszel-socket`).

**Never define a separate network inside a per-app compose fragment.** Everything shares `vps_network` via the root compose default.

## MediaFlow + Warp

MediaFlow routes all traffic through Warp (`caomingjun/warp`) so debrid add-ons (Torrentio, etc.) aren't blocked by debrid providers.

- Warp exposes an HTTP proxy on port `1080`
- Per-domain proxy routing is configured explicitly inside `apps/mediaflow-proxy/config.toml`
- Warp requires `cap_add: NET_ADMIN` and the `src_valid_mark` sysctl
- MediaFlow uses `depends_on: warp: condition: service_healthy` — Warp must confirm `warp=on` before MediaFlow starts
- `warp-data` volume persists the registration across restarts

Warp lives in `apps/warp/` as a standalone service. MediaFlow depends on it but they are separate compose fragments.

## Conventions

- All compose files define `deploy.resources.limits` + `reservations`, `stop_grace_period`, and `logging`
- Healthchecks on all services where meaningful
- `restart: unless-stopped` everywhere
- No hardcoded secrets — `.env` only
- `image: ...:latest` is intentional; Watchtower handles updates

## What to avoid

- `ports:` on internal services
- Separate networks in per-app compose files
- Docker socket mounted read-write except on Watchtower
- Data persisted inside containers

## Common operations

```bash
# Start
docker compose up -d

# Status + memory
docker compose ps
docker stats --no-stream

# Reload Caddy
docker compose exec -w /etc/caddy caddy caddy reload

# Logs
docker compose logs -f <service>

# Update all images
docker compose pull && docker compose up -d

# Verify Warp tunnel
docker compose exec warp curl -s https://cloudflare.com/cdn-cgi/trace | grep warp
```
