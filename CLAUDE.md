# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS. All services run behind Caddy (reverse proxy + automatic TLS). One `.env` file drives the entire stack.

## Stack

| Service | Image | Role |
|---|---|---|
| AdGuard Home | `adguard/adguardhome:latest` | DNS + ad blocking |
| AIOMetadata | `ghcr.io/cedya77/aiometadata:latest` | Stremio catalog & metadata generator |
| AIOMetadata Cache | `redis:alpine` | Redis for AIOMetadata |
| AIOStreams | `ghcr.io/viren070/aiostreams:latest` | Stremio addon aggregator |
| Beszel | `henrygd/beszel:latest` | Monitoring dashboard |
| Beszel Agent | `henrygd/beszel-agent:latest` | Host metrics (`network_mode: host`) |
| Caddy | `caddy:latest` | Reverse proxy, TLS |
| Dispatcharr | `ghcr.io/dispatcharr/dispatcharr:latest` | IPTV stream management (web) |
| Dispatcharr Celery | `ghcr.io/dispatcharr/dispatcharr:latest` | IPTV stream management (workers) |
| Filebrowser | `filebrowser/filebrowser:v2-alpine` | Web file manager (serves `/opt`) |
| Ghostfolio | `ghostfolio/ghostfolio:latest` | Portfolio tracker |
| Ghostfolio Cache | `redis:alpine` | Redis for Ghostfolio |
| Ghostfolio DB | `postgres:16-alpine` | Postgres for Ghostfolio |
| Honey | `ghcr.io/dani3l0/honey:latest` | Dashboard / start page |
| MediaFlow Proxy | `ghcr.io/mhdzumair/mediaflow-proxy-light:latest` | Debrid media proxy |
| Stirling-PDF | `stirlingtools/stirling-pdf:latest` | PDF manipulation tools |
| Uptime Kuma | `louislam/uptime-kuma:latest` | Uptime monitoring & status page |
| Wallos | `bellamy/wallos:latest` | Subscription tracker |
| Warp | `caomingjun/warp:latest` | Cloudflare HTTP proxy for MediaFlow |
| Watchtower | `nickfedor/watchtower:latest` | Auto image updates |
| WG-Easy | `ghcr.io/wg-easy/wg-easy:latest` | WireGuard VPN server + web UI |

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
    config.toml       # mediaflow-proxy only
```

Data is persisted under `$DOCKER_DATA_DIR` (default `/opt/docker/data`), never in the repo.

## Network

All services share `vps_network` (defined in root `compose.yaml`). Services talk to each other by container name.

Only Caddy and AdGuard expose host ports (`80`, `443`, `53`, `853`). All other services use `expose:` only — never `ports:`.

Beszel Agent uses `network_mode: host` and communicates with the hub via a Unix socket volume (`beszel-socket`).

**Never define a separate network inside a per-app compose fragment.** Everything shares `vps_network` via the root compose default.

## Dispatcharr

Dispatcharr runs as a single container `dispatcharr` in All-in-One (`aio`) mode (`DISPATCHARR_ENV=aio`). It manages the Django web application, background celery workers, and broker internally. It persists data at `$DOCKER_DATA_DIR/dispatcharr`.

## MediaFlow + Warp

MediaFlow routes all traffic through Warp (`caomingjun/warp`) so debrid add-ons (Torrentio, etc.) aren't blocked by debrid providers.

- Warp exposes an HTTP proxy on port `1080`
- Per-domain proxy routing is configured explicitly inside `apps/mediaflow-proxy/config.toml`
- Warp requires `cap_add: NET_ADMIN` and the `src_valid_mark` sysctl
- MediaFlow uses `depends_on: warp: condition: service_healthy` — Warp must confirm `warp=on` before MediaFlow starts
- `warp-data` volume persists the registration across restarts

## Caddy

The Caddyfile defines a `(common)` snippet applied to every vhost:

```
(common) {
    encode zstd gzip
}
```

HTTP/3 (QUIC) is enabled globally. All subdomains are derived from `.env` variables and injected via `environment:` in the Caddy compose fragment.

## Conventions

- `stop_grace_period` and `logging` on all services
- Healthchecks on all services where meaningful
- `restart: always` on critical services (Caddy, Warp, AdGuard, MediaFlow, Dispatcharr)
- `restart: unless-stopped` on personal tools (Ghostfolio, Wallos, Honey, Stirling-PDF, etc.)
- No hardcoded secrets — `.env` only
- `image: ...:latest` is intentional; Watchtower handles updates
- No `deploy.resources` limits — let the host scheduler manage allocation
- **Alphabetical Sorting:** Keep `.env`/`.env.example` subdomains, root `compose.yaml` includes, `apps/caddy/compose.yaml` environment variables, `apps/caddy/Caddyfile` reverse proxy blocks, and `apps/honey/config.json` services list sorted alphabetically.

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
