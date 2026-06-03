# VPS Stack

Self-hosted Docker stack for a 2 GB VPS. One `.env` file drives everything.

## Services

| Container | Purpose | URL |
|---|---|---|
| Caddy | Reverse proxy + TLS | — |
| AdGuard Home | DNS + ad blocking | `adguard.$DOMAIN` |
| AIOMetadata | Stremio catalog enrichment | `aiometadata.$DOMAIN` |
| AIOStreams | Stremio addon aggregator | `aiostreams.$DOMAIN` |
| Beszel | Resource monitoring | `beszel.$DOMAIN` |
| Beszel Agent | Host metrics collector | — |
| Dispatcharr | IPTV stream management | `dispatcharr.$DOMAIN` |
| Filebrowser | Web file manager | `files.$DOMAIN` |
| Ghostfolio | Portfolio tracker | `ghostfolio.$DOMAIN` |
| Honey | Dashboard | `$DOMAIN` |
| MediaFlow Proxy | Debrid media proxy | `mediaflow.$DOMAIN` |
| Warp | Cloudflare proxy for MediaFlow | — |
| Watchtower | Auto-updates | — |
| Zublo | Subscription tracker | `zublo.$DOMAIN` |

---

## Setup

### 1. Host preparation

```bash
sudo apt-get update && sudo apt-get install -y curl git
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
```

**Free port 53 for AdGuard:**

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

**Swap (required on 2 GB VPS):**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p
```

**System tuning:**

```bash
echo 'vm.overcommit_memory=1' | sudo tee -a /etc/sysctl.conf && sudo sysctl -p

sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=100M\nRuntimeMaxUse=50M\n' | \
  sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

**Docker log rotation:**

```bash
sudo cp daemon.json /etc/docker/daemon.json
sudo systemctl restart docker
```

---

### 2. Configure

```bash
sudo git clone https://github.com/guclusefa/vps.git /opt/docker
sudo chown -R $USER:$USER /opt/docker
cd /opt/docker
cp .env.example .env
```

Edit `.env` and fill in your domain and secrets. Generate each secret with:

```bash
openssl rand -hex 32
```

---

### 3. First launch

AdGuard's setup wizard runs on port `3000`. Before starting, open `apps/caddy/Caddyfile` and temporarily change the AdGuard line:

```
reverse_proxy adguard:3000
```

Start the stack:

```bash
docker compose up -d
```

Open `https://adguard.$DOMAIN` and complete the wizard. Set:
- **Web interface port:** `80`
- **DNS server port:** `53`

Once done, restore the Caddyfile to `adguard:80` and reload:

```bash
docker compose exec -w /etc/caddy caddy caddy reload
```

---

### 4. Beszel agent

After first launch:

1. Open `https://beszel.$DOMAIN` → create admin account → **Add System**
2. Copy the `KEY` and `TOKEN` into `.env`
3. Restart the agent: `docker compose up -d beszel-agent`
4. Set **Host / IP** to `/beszel_socket/beszel.sock`

---

### 5. AdGuard DNS-over-TLS (optional)

Enables `adguard.$DOMAIN` as a Private DNS provider on mobile.

In AdGuard → **Settings → Encryption settings**:

| Field | Value |
|---|---|
| Enable encryption | ✓ |
| Server name | `adguard.$DOMAIN` |
| Certificate path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.crt` |
| Private key path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.key` |

On your phone:
- **Android:** Settings → Private DNS → enter `adguard.$DOMAIN`
- **iOS:** AdGuard UI → Setup Guide → install DNS profile

> Run `find /opt/docker/data/caddy/data -name "*.crt"` to confirm the exact certificate path on your server.

---

## Verify

```bash
docker compose ps
docker stats --no-stream
```
