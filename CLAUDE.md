# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS (DigitalOcean, 1 vCPU, 2 GB RAM).
All services run behind Caddy (reverse proxy + automatic TLS).
One `.env` file drives the entire stack.

## Stack

| Service | Image | Role |
|---|---|---|
| Caddy | `caddy:latest` | Reverse proxy, TLS |
| AdGuard Home | `adguard/adguardhome:latest` | DNS + ad blocking |
| Beszel | `henrygd/beszel:latest` | Monitoring dashboard |
| Beszel Agent | `henrygd/beszel-agent:latest` | Host metrics (`network_mode: host`) |
| Dispatcharr | `ghcr.io/dispatcharr/dispatcharr:latest` | IPTV stream management |
| Filebrowser | `filebrowser/filebrowser:v2-alpine` | Web file manager (serves `/opt`) |
| Ghostfolio | `ghostfolio/ghostfolio:latest` | Portfolio tracker |
| Ghostfolio DB | `postgres:15-alpine` | Postgres for Ghostfolio |
| Ghostfolio Cache | `redis:alpine` | Redis for Ghostfolio |
| Honey | `ghcr.io/dani3l0/honey:latest` | Dashboard / start page |
| MediaFlow Proxy | `ghcr.io/mhdzumair/mediaflow-proxy-light:latest` | Debrid media proxy |
| Warp | `caomingjun/warp:latest` | Cloudflare HTTP proxy for MediaFlow |
| Watchtower | `nickfedor/watchtower:latest` | Auto image updates |
| Zublo | `ghcr.io/danielalves96/zublo:latest` | Subscription tracker |

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

## Memory budget

Hard limits across all 14 containers. Keep total under ~2000M.

| Container | Limit |
|---|---|
| Caddy | 128M |
| AdGuard Home | 192M |
| Beszel | 64M |
| Beszel Agent | 64M |
| Dispatcharr | 768M |
| Filebrowser | 32M |
| Ghostfolio | 384M |
| Ghostfolio DB | 64M |
| Ghostfolio Cache | 32M |
| Honey | 48M |
| MediaFlow Proxy | 96M |
| Warp | 64M |
| Watchtower | 64M |
| Zublo | 64M |
| **Total** | **2068M** |

## MediaFlow + Warp

MediaFlow routes all traffic through Warp (`caomingjun/warp`) so debrid add-ons (Torrentio, etc.) aren't blocked by debrid providers.

- Warp exposes an HTTP proxy on port `1080`
- `APP__PROXY__ALL_PROXY=true` sends all MediaFlow traffic through it — no per-domain routing needed
- Warp requires `cap_add: NET_ADMIN` and the `src_valid_mark` sysctl
- MediaFlow uses `depends_on: warp: condition: service_healthy` — Warp must confirm `warp=on` before MediaFlow starts
- `warp-data` volume persists the registration across restarts

Warp lives in `apps/warp/` as a standalone service. MediaFlow depends on it but they are separate compose fragments.

## Dispatcharr

- AIO mode (`DISPATCHARR_ENV=aio`): Redis + Celery + Daphne + uWSGI in one container
- `uwsgi.ini` → mounted at `/app/docker/uwsgi.ini`
- `redis.conf` → mounted at `/etc/dispatcharr/redis.conf` (64M cap, LRU, no persistence)
- `workers = 1` is intentional — raising to 2 adds ~150M
- `post-buffering = 0` is required for IPTV streaming — do not change
- `gevent = 50` — do not raise without profiling

## Conventions

- All compose files define `deploy.resources.limits` + `reservations`, `stop_grace_period`, and `logging`
- Healthchecks on all services where meaningful
- `restart: unless-stopped` everywhere
- No hardcoded secrets — `.env` only
- `image: ...:latest` is intentional; Watchtower handles updates

## What to avoid

- `ports:` on internal services
- Separate networks in per-app compose files
- Raising `gevent` above 50
- A second Redis container (Dispatcharr embeds its own)
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
