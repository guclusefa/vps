# CLAUDE.md

Context and conventions for AI-assisted work on this repo.

## What this repo is

A self-hosted Docker stack for a personal VPS. All services run behind Caddy (reverse proxy + automatic TLS). One [.env](file:///home/ceest/dev/vps/.env) file drives the entire stack.

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

All services share `vps_network` (defined in root [compose.yaml](file:///home/ceest/dev/vps/compose.yaml)). Services talk to each other by container name.

Only Caddy and AdGuard expose host ports (`80`, `443`, `53`, `853`). All other services rely on Docker network service resolution — never use `ports:` or `expose:`.

Beszel Agent uses `network_mode: host` and communicates with the hub via a Unix socket volume (`beszel-socket`).

**Never define a separate network inside a per-app compose fragment.** Everything shares `vps_network` via the root compose default.

## Dispatcharr

Dispatcharr runs as a single container `dispatcharr` in All-in-One (`aio`) mode (`DISPATCHARR_ENV=aio`). It manages the Django web application, background celery workers, and broker internally. It persists data at `$DOCKER_DATA_DIR/dispatcharr`.

## MediaFlow + Warp

MediaFlow routes all traffic through Warp (`caomingjun/warp`) so debrid add-ons (Torrentio, etc.) aren't blocked by debrid providers.

- Warp exposes an HTTP proxy on port `1080`
- Per-domain proxy routing is configured explicitly inside `apps/mediaflow-proxy/config.toml`
- Warp requires `cap_add: NET_ADMIN` and the `src_valid_mark` sysctl
- MediaFlow uses `depends_on: [warp]` — starts after Warp container starts
- `warp-data` volume persists the registration across restarts

## Caddy

The Caddyfile defines a `(common)` snippet applied to every vhost:

```
(common) {
    encode zstd gzip
}
```

HTTP/3 (QUIC) is enabled globally. All subdomains are derived from [.env](file:///home/ceest/dev/vps/.env) variables and injected via `environment:` in the Caddy compose fragment.

## Conventions

- `logging` is handled globally by `daemon.json`; custom `stop_grace_period` is used only when overriding the default `10s`.
- No custom healthchecks (rely on image defaults)
- `restart: always` on critical services (Caddy, Warp, AdGuard, MediaFlow, Dispatcharr, WG-Easy)
- `restart: unless-stopped` on personal tools (Ghostfolio, Wallos, Honey, Stirling-PDF, Comet, etc.)
- No hardcoded secrets — [.env](file:///home/ceest/dev/vps/.env) only
- `image: ...:latest` is intentional; Watchtower handles updates
- No `deploy.resources` limits — let the host scheduler manage allocation
- **Alphabetical Sorting:** Keep [.env](file:///home/ceest/dev/vps/.env)/[.env.example](file:///home/ceest/dev/vps/.env.example) subdomains, root [compose.yaml](file:///home/ceest/dev/vps/compose.yaml) includes, [apps/caddy/compose.yaml](file:///home/ceest/dev/vps/apps/caddy/compose.yaml) environment variables, [apps/caddy/Caddyfile](file:///home/ceest/dev/vps/apps/caddy/Caddyfile) reverse proxy blocks, and [apps/honey/config.json](file:///home/ceest/dev/vps/apps/honey/config.json) services list sorted alphabetically.

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
