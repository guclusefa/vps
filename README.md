# VPS Stack

Self-hosted Docker stack with Caddy reverse proxy and automatic TLS. Services include AdGuard Home, Stremio addons (AIOStreams, Comet, MediaFusion), a portfolio tracker, Stirling-PDF, WireGuard VPN, and more. A single `.env` file configures the entire stack.

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

**Swap:**

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

**Firewall:**

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 443/udp
sudo ufw allow 853/tcp
sudo ufw allow 51820/udp
sudo ufw --force enable
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

AdGuard's setup wizard runs on port `3000`. Before starting, open `.env` and temporarily set the port:

```ini
ADGUARD_PORT=3000
```

Start the stack:

```bash
docker compose up -d
```

Open `https://adguard.$DOMAIN` and complete the wizard. Set:

- **Web interface port:** `80`
- **DNS server port:** `53`

Once done, restore `ADGUARD_PORT` to `80` in `.env` and apply the changes by recreating Caddy:

```bash
docker compose up -d caddy
```

---

### 4. Beszel agent

After first launch:

1. Open `https://beszel.$DOMAIN` → create admin account → **Add System**
2. Copy the `KEY` and `TOKEN` into `.env`
3. Restart the agent: `docker compose up -d beszel-agent`
4. Set **Host / IP** to `/beszel_socket/beszel.sock`

---

### 5. AdGuard DNS-over-TLS (Required for External Devices)

Enables `adguard.$DOMAIN` as a Private DNS provider.

In AdGuard → **Settings → Encryption settings**:

| Field | Value |
|---|---|
| Enable encryption | ✓ |
| Server name | `adguard.$DOMAIN` |
| Certificate path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.crt` |
| Private key path | `/etc/caddy-certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/adguard.$DOMAIN/adguard.$DOMAIN.key` |

---

## Verify

```bash
docker compose ps
docker stats --no-stream
```
