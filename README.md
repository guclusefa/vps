# VPS Self-Hosted Stack

A single `.env` file drives the entire stack. Optimized for 2 GB RAM VPS.

## Services & Memory Limits

| Container | Purpose | URL | Limit |
|---|---|---|---|
| Caddy | Reverse proxy + TLS | — | 128M |
| AdGuard Home | DNS + ad blocking | `adguard.$DOMAIN` | 192M |
| Beszel | Resource monitoring | `beszel.$DOMAIN` | 64M |
| Beszel Agent | Metrics collector | — | 64M |
| Dispatcharr | Stream management | `dispatcharr.$DOMAIN` | 768M |
| Filebrowser | Web file manager | `files.$DOMAIN` | 32M |
| Ghostfolio | Portfolio tracker | `ghostfolio.$DOMAIN` | 384M |
| Ghostfolio DB | Postgres storage | — | 64M |
| Ghostfolio Cache | Redis cache | — | 32M |
| Honey | Dashboard | `$DOMAIN` | 48M |
| MediaFlow Proxy | Debrid media proxy | `mediaflow.$DOMAIN` | 96M |
| Watchtower | Auto-updates | — | 64M |
| Zublo | Finance tracker | `zublo.$DOMAIN` | 64M |
| **Total** | | | **1936M** |

Real-world usage sits around 750–1050M. The gap gives headroom for traffic spikes without OOM kills.

---

## 1. Host Preparation

```bash
# Update and install prerequisites
sudo apt-get update && sudo apt-get install -y curl git

# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the docker group (re-login after this)
sudo usermod -aG docker $USER
```

### Free port 53 for AdGuard

```bash
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
```

### Swap (required on 2 GB VPS)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Reduce swap aggressiveness (keep hot data in RAM)
echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### System tuning

```bash
# Avoid false OOM kills from overcommit accounting
echo 'vm.overcommit_memory=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Cap journald size (prevents log creep on low-disk VPS)
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nSystemMaxUse=100M\nRuntimeMaxUse=50M\n' | \
  sudo tee /etc/systemd/journald.conf.d/size.conf
sudo systemctl restart systemd-journald
```

### Docker log rotation

Cap container log size at the daemon level so it applies to every container automatically:

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'DAEMON'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
DAEMON
sudo systemctl restart docker
```

> Each compose file also sets `logging:` per-service as a belt-and-suspenders fallback.

---

## 2. Clone & Configure

```bash
sudo git clone https://github.com/guclusefa/vps.git /opt/docker
sudo chown -R $USER:$USER /opt/docker
cd /opt/docker

cp .env.example .env
nano .env  # Fill in your domain, passwords, and paths
```

Generate secrets before starting:

```bash
# Ghostfolio and Zublo each need a unique secret
openssl rand -hex 32  # repeat for each key — do not reuse the same value
```

Paste the output into `GHOSTFOLIO_ACCESS_TOKEN_SALT`, `GHOSTFOLIO_JWT_SECRET_KEY`, and `ZUBLO_PB_ENCRYPTION_KEY` in `.env`.

---

## 3. First Launch (AdGuard setup wizard)

AdGuard uses port 3000 for its initial setup wizard. Temporarily point Caddy to it:

```bash
# In apps/caddy/Caddyfile, set AdGuard to port 3000:
#   reverse_proxy adguard:3000

docker compose up -d
```

Then open `https://adguard.$DOMAIN` and complete the wizard:
- **Web interface port:** `80`
- **DNS server port:** `53`

---

## 4. Production Lockdown

Once the AdGuard wizard is done, switch Caddy back to port 80:

```bash
# In apps/caddy/Caddyfile, restore:
#   reverse_proxy adguard:80

docker compose exec -w /etc/caddy caddy caddy reload
```

---

## 5. Beszel Setup (monitoring)

Beszel requires a one-time step to connect the agent after first launch.

1. Open `https://beszel.$DOMAIN` and create your admin account.
2. Click **Add System** — the UI will show you a `KEY` and `TOKEN`.
3. Copy them into your `.env`:
```env
BESZEL_AGENT_KEY=<key from UI>
BESZEL_AGENT_TOKEN=<token from UI>
```
4. Restart the agent:
```bash
docker compose up -d beszel-agent
```
5. Back in the UI, set **Host / IP** to:
```
/beszel_socket/beszel.sock
```

---

## Verify

```bash
docker compose ps
docker stats --no-stream
```

Healthy output: all containers `Up (healthy)` where applicable, no container near its memory limit, Dispatcharr below 700M.
