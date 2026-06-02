# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS (DigitalOcean, 1 vCPU, 2 GB RAM).
All services live behind Caddy (reverse proxy + automatic TLS via DuckDNS).
One `.env` file drives the entire stack.

## Stack overview

| Service | Image | Role |
|---|---|---|
| Caddy | `caddy:latest` | Reverse proxy, TLS termination |
| AdGuard Home | `adguard/adguardhome:latest` | DNS resolver + ad blocking |
| Beszel | `henrygd/beszel:latest` | Resource monitoring dashboard |
| Beszel Agent | `henrygd/beszel-agent:latest` | Host metrics collector (`network_mode: host`) |
| Dispatcharr | `ghcr.io/dispatcharr/dispatcharr:latest` | IPTV stream management |
| Filebrowser | `filebrowser/filebrowser:v2-alpine` | Web-based file manager (serves `/opt`) |
| Ghostfolio | `ghostfolio/ghostfolio:latest` | Portfolio & wealth management |
| Ghostfolio DB | `postgres:15-alpine` | Postgres storage for Ghostfolio |
| Ghostfolio Cache | `redis:alpine` | Redis cache for Ghostfolio |
| Honey | `ghcr.io/dani3l0/honey:latest` | Start page / dashboard |
| MediaFlow Proxy | `ghcr.io/mhdzumair/mediaflow-proxy-light:latest` | Debrid media proxy (Rust, light edition) |
| Watchtower | `nickfedor/watchtower:latest` | Automatic image updates |
| Zublo | `ghcr.io/danielalves96/zublo:latest` | Personal finance tracker |

## Repo structure

```
compose.yaml              # Root: includes all apps via `include:`
.env.example              # Template — copy to .env and fill
.gitattributes            # Enforces LF line endings across the repo
.gitignore
CLAUDE.md                 # This file
README.md                 # Setup guide
apps/
  <service>/
    compose.yaml          # Per-service compose fragment
    Caddyfile             # Caddy only
    config.json           # Honey only
    redis.conf            # Dispatcharr only (custom Redis config for AIO mode)
    uwsgi.ini             # Dispatcharr only (custom uWSGI config for AIO mode)
```

Data is persisted under `$DOCKER_DATA_DIR` (default `/opt/docker/data`), never inside the repo.

## Network

All services share a single user-defined bridge network (`vps_network`, defined in root `compose.yaml`).
Services communicate by container name (e.g. `dispatcharr:9191`).
Only Caddy and AdGuard expose ports to the host (`80`, `443`, `53`).
Never add `ports:` to internal services — use `expose:` only.

The Beszel Agent is an exception: it uses `network_mode: host` to collect host-level metrics.
It communicates with the Beszel hub via a shared Unix socket volume (`beszel-socket`), bypassing the bridge network entirely.

## Memory budget

Total hard limit: **1936M** across all 13 containers. The ceiling is kept under **~1950M** to guarantee host OS stability on a 2 GB node.

| Container | Limit |
|---|---|
| Caddy | 128M |
| AdGuard Home | 128M |
| Beszel | 64M |
| Beszel Agent | 64M |
| Dispatcharr | 768M |
| Filebrowser | 32M |
| Ghostfolio | 384M |
| Ghostfolio DB | 64M |
| Ghostfolio Cache | 32M |
| Honey | 48M |
| MediaFlow Proxy | 96M |
| Watchtower | 64M |
| Zublo | 64M |
| **Total** | **1936M** |

Do not raise a limit without verifying the running total stays under ~1950M.
Dispatcharr is capped at 768M — optimized by mounting a custom `uwsgi.ini` and `redis.conf`.

## Dispatcharr specifics

- Runs in AIO mode (`DISPATCHARR_ENV=aio`): Redis + Celery workers + Daphne + uWSGI in a single container.
- `uwsgi.ini` in `apps/dispatcharr/` is mounted into the AIO container at `/app/docker/uwsgi.ini`.
- `redis.conf` in `apps/dispatcharr/` is mounted at `/etc/dispatcharr/redis.conf` and caps Redis at 64 MB with LRU eviction and no persistence.
- `workers = 1` is intentional for single-user use. Raising to 2 doubles the Django/gevent footprint (~+150M).
- `post-buffering = 0` disables request buffering — required for real-time IPTV streaming. Do not set a non-zero value.
- Streaming timeouts are intentionally long (`http-timeout = 600`, `socket-timeout = 600`).
- gevent is used for concurrency (`gevent = 50`). Monkey-patching is required and already configured.

## Conventions

- All compose files define `deploy.resources.limits` and `deploy.resources.reservations`.
- All compose files define `stop_grace_period` for clean shutdowns.
- All compose files define `logging` with `max-size`/`max-file` (belt-and-suspenders on top of `daemon.json`).
- Healthchecks are defined on critical services: Caddy, AdGuard, Dispatcharr, Ghostfolio stack.
- `restart: unless-stopped` on all services.
- No hardcoded secrets — everything via `.env`.
- `image: ...:latest` is intentional; Watchtower handles updates. Consider pinning dispatcharr to a version tag if upstream breaks.

## What to avoid

- Don't add `ports:` to services that only need to be reachable via Caddy.
- Don't raise `gevent` above 50 without profiling.
- Don't add a second Redis container — Dispatcharr embeds its own in AIO mode.
- Don't mount the Docker socket read-write except for Watchtower. Beszel Agent uses `:ro`.
- Don't persist data inside the container — always volume-mount to `$DOCKER_DATA_DIR/<service>`.
- Don't commit `.env` — only `.env.example` goes in git.

## Common operations

```bash
# Start everything
docker compose up -d

# Check status and memory
docker compose ps
docker stats --no-stream

# Reload Caddy config without restart
docker compose exec -w /etc/caddy caddy caddy reload

# Follow logs for a service
docker compose logs -f dispatcharr

# Pull latest images and recreate changed containers
docker compose pull && docker compose up -d

# Full teardown (keeps volumes)
docker compose down
```
