# Coolify Self-Hosted Setup on GuildServer

Install Coolify PaaS on the `usher-node` server (`143.105.102.121:5555`) and expose it via Cloudflare Tunnel at `coolify.guildserver.io`, without disrupting existing services.

## User Review Required

> [!IMPORTANT]
> **Cloudflare API Token Required**: You'll need a Cloudflare API token with Zone:DNS:Edit permissions for `guildserver.io`, or access to the Cloudflare dashboard to create the tunnel and DNS records manually.

> [!WARNING]
> **Coolify uses ports 80, 443, and 8000 on the host**. Coolify's built-in Traefik proxy binds to ports 80 and 443. The Coolify dashboard itself runs on port 8000. If any existing services use these ports, we'll need to address conflicts before installing. Since we're using a Cloudflare Tunnel, we can configure Coolify to **not** bind 80/443 publicly and route only through the tunnel—but the default install binds them. We will handle this.

> [!CAUTION]
> **SSH is on port 5555, not 22**. The Coolify install script expects SSH on port 22 by default for localhost connectivity. We'll configure Coolify to use port 5555 for the localhost server connection post-install.

## Open Questions

> [!IMPORTANT]
> 1. **Do you already have a `cloudflared` tunnel running on this server?** From previous conversations about `guildserver.io`, you may already have a Cloudflare tunnel configured. If so, we just need to add a new public hostname entry. If not, we'll create a new tunnel.
> 2. **Is there an existing `cloudflared` daemon/service on the server?** This determines whether we install `cloudflared` fresh or reuse the existing setup.
> 3. **Are ports 80, 443 already in use by another service (e.g., Nginx, Traefik, Caddy)?** This affects whether we need to change Coolify's Traefik binding ports.

---

## Proposed Changes

### Phase 1: Server Reconnaissance

Before touching anything, audit the server to understand what's running.

#### Commands to run on the server

```bash
# SSH into the server
ssh -p 5555 usher-node@143.105.102.121

# Check OS version (Coolify requires Linux, Ubuntu LTS recommended)
cat /etc/os-release

# Check available disk space (Coolify needs 30GB total, 20GB available)
df -BG /

# Check RAM and CPU
free -h && nproc

# Check if Docker is already installed
docker --version 2>/dev/null || echo "Docker not installed"

# Check what's using ports 80, 443, 8000
sudo ss -tlnp | grep -E ':80 |:443 |:8000 '

# Check running Docker containers
docker ps 2>/dev/null || echo "Docker not running"

# Check if cloudflared is already installed/running
which cloudflared 2>/dev/null && cloudflared --version || echo "cloudflared not found"
systemctl status cloudflared 2>/dev/null || echo "cloudflared service not found"

# Check existing Cloudflare tunnel config
cat /etc/cloudflared/config.yml 2>/dev/null || cat ~/.cloudflared/config.yml 2>/dev/null || echo "No cloudflared config found"

# Check what's in /data/coolify (if anything)
ls -la /data/coolify 2>/dev/null || echo "No existing Coolify installation"
```

---

### Phase 2: Pre-Installation — Port Conflict Resolution

Based on the reconnaissance, if ports 80/443 are in use:

**Option A** (Recommended if no other web server is on 80/443): Let Coolify own ports 80/443 via its Traefik proxy. Cloudflare Tunnel will connect to `http://localhost:80` for wildcard resources and `http://localhost:8000` for the dashboard.

**Option B** (If another web server is already on 80/443): Change Coolify's Traefik ports to non-conflicting ones (e.g., 8080/8443) by editing `/data/coolify/source/docker-compose.yml` after installation.

---

### Phase 3: Install Coolify

#### Step 3.1 — Run the official install script as root

```bash
# Become root
sudo su -

# Run the official Coolify install script
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | sudo bash
```

**What the script does (9 steps):**
1. Installs required packages: `curl`, `wget`, `git`, `jq`, `openssl`
2. Checks/installs OpenSSH server
3. Installs Docker if not present (requires Docker 24+)
4. Configures Docker daemon (`/etc/docker/daemon.json` — log settings, address pools)
5. Downloads Coolify compose files to `/data/coolify/source/`
6. Creates/merges `.env` file with generated secrets
7. Sets up environment variables (APP_KEY, DB_PASSWORD, REDIS_PASSWORD, etc.)
8. Generates SSH key for localhost access
9. Pulls and starts Coolify containers via `docker compose`

**After install, these containers will be running:**
- `coolify` — Main Laravel app (port 8000)
- `coolify-db` — PostgreSQL database
- `coolify-redis` — Redis cache
- `coolify-realtime` — WebSocket server (port 6001)
- `coolify-proxy` — Traefik reverse proxy (ports 80, 443)

#### Step 3.2 — Verify installation

```bash
# Check all containers are running
docker ps --filter "name=coolify"

# Check Coolify health
docker inspect --format='{{.State.Health.Status}}' coolify

# Test local access
curl -s http://localhost:8000 | head -20
```

---

### Phase 4: Configure Coolify's SSH Port for Localhost

Coolify connects to "localhost" via SSH to manage the server it's installed on. Since SSH runs on port 5555, not 22:

```bash
# After first login to Coolify UI (step 5), go to:
# Settings → Servers → localhost → edit SSH Port to 5555
# Then click "Validate Server" to verify connectivity
```

Alternatively, if Coolify's localhost SSH connection fails during onboarding, you can fix it post-install.

---

### Phase 5: Cloudflare Tunnel Setup for `coolify.guildserver.io`

This follows the [Coolify Cloudflare Tunnels — All Resources](https://coolify.io/docs/integrations/cloudflare/tunnels/all-resource) guide.

#### Step 5.1 — Install `cloudflared` (if not already installed)

```bash
# On Debian/Ubuntu:
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
rm cloudflared.deb

# Verify
cloudflared --version
```

#### Step 5.2 — Authenticate and create tunnel (if no existing tunnel)

**If creating a new tunnel:**

```bash
# Login to Cloudflare (opens browser or provides URL)
cloudflared tunnel login

# Create a tunnel named "guildserver"
cloudflared tunnel create guildserver

# Note the tunnel UUID from the output — you'll need it
```

**If reusing an existing tunnel:**
Just add the new public hostname entry in step 5.3.

#### Step 5.3 — Configure the tunnel for Coolify dashboard

**Option A — Via Cloudflare Dashboard (Zero Trust):**
1. Go to https://one.dash.cloudflare.com → Networks → Tunnels
2. Select your tunnel (or the new "guildserver" tunnel)
3. Add Public Hostname:
   - **Subdomain**: `coolify`
   - **Domain**: `guildserver.io`
   - **Service**: `http://localhost:8000`
4. Under **Additional application settings → TLS → No TLS Verify**: Enable (since the backend is HTTP, not HTTPS)

**Option B — Via CLI config file (`/etc/cloudflared/config.yml`):**

```yaml
# Add this ingress rule to your existing config
# If creating from scratch:
tunnel: <TUNNEL_UUID>
credentials-file: /root/.cloudflared/<TUNNEL_UUID>.json

ingress:
  # Coolify Dashboard
  - hostname: coolify.guildserver.io
    service: http://localhost:8000
  # Wildcard for Coolify-deployed apps (optional, for future)
  # - hostname: "*.coolify.guildserver.io"
  #   service: http://localhost:80
  # Catch-all (required)
  - service: http_status:404
```

#### Step 5.4 — Create DNS CNAME record

```bash
# If using CLI:
cloudflared tunnel route dns <TUNNEL_UUID_OR_NAME> coolify.guildserver.io

# Or manually in Cloudflare Dashboard:
# Type: CNAME
# Name: coolify
# Target: <TUNNEL_UUID>.cfargotunnel.com
# Proxy: Yes (orange cloud)
```

#### Step 5.5 — Start/restart the tunnel

```bash
# If running as a systemd service:
sudo systemctl restart cloudflared

# Or install as service for the first time:
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared

# Verify the tunnel is running:
cloudflared tunnel info <TUNNEL_NAME>
```

#### Step 5.6 — Cloudflare SSL/TLS settings

In the Cloudflare Dashboard for `guildserver.io`:
1. Go to **SSL/TLS** → Set encryption mode to **Flexible** (since Coolify dashboard serves HTTP on port 8000)
   - OR set to **Full** if you configure Coolify with HTTPS (advanced)
2. Under **SSL/TLS → Edge Certificates**, ensure "Always Use HTTPS" is ON

---

### Phase 6: First-Time Coolify Setup

1. Open `https://coolify.guildserver.io` in your browser
2. Create root admin account (email + password)
3. **CRITICAL**: Go to **Settings → Servers → localhost**:
   - Change SSH port from `22` to `5555`
   - Click "Validate Server" — should show green/healthy
4. Back up the `.env` file:
   ```bash
   cp /data/coolify/source/.env /data/coolify/source/.env.backup
   ```

---

### Phase 7: Post-Install Hardening (Optional but Recommended)

#### 7.1 — Disable direct port 8000 access (since tunnel handles it)

If you don't want the Coolify dashboard accessible on the raw IP:

```bash
# Using iptables (if not using ufw):
sudo iptables -A INPUT -p tcp --dport 8000 -j DROP
sudo iptables -I INPUT -p tcp --dport 8000 -s 127.0.0.1 -j ACCEPT

# Or using ufw:
sudo ufw deny 8000
```

#### 7.2 — Restrict Coolify's Traefik ports (if not needed publicly)

If you only serve apps through the Cloudflare Tunnel and don't need ports 80/443 publicly:

```bash
# Block 80/443 from external but allow localhost (for tunnel)
sudo iptables -A INPUT -p tcp --dport 80 -s 127.0.0.1 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j DROP
sudo iptables -A INPUT -p tcp --dport 443 -s 127.0.0.1 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j DROP
```

---

## Verification Plan

### Automated Tests

```bash
# 1. Check all Coolify containers are running and healthy
docker ps --filter "name=coolify" --format "table {{.Names}}\t{{.Status}}"

# 2. Verify Coolify health endpoint
curl -s http://localhost:8000 | grep -i coolify

# 3. Verify tunnel is active
cloudflared tunnel info <TUNNEL_NAME>

# 4. Test external access via the subdomain
curl -sI https://coolify.guildserver.io

# 5. Verify no port conflicts with existing services
docker ps --format "table {{.Names}}\t{{.Ports}}" | head -20
```

### Manual Verification

- Open `https://coolify.guildserver.io` in a browser → should show Coolify registration/login page
- Complete the onboarding wizard
- Validate the localhost server connection (SSH on port 5555)
- Deploy a test application to confirm full functionality

---

## Summary of Files/Paths Affected on Server

| Path | Purpose |
|------|---------|
| `/data/coolify/` | All Coolify data (source, DBs, apps, backups, proxy) |
| `/data/coolify/source/.env` | Coolify environment config (secrets, keys) — **BACK THIS UP** |
| `/data/coolify/source/docker-compose.yml` | Docker Compose file for Coolify stack |
| `/etc/docker/daemon.json` | Docker daemon configuration |
| `/etc/cloudflared/config.yml` | Cloudflare tunnel configuration |
| `/root/.cloudflared/` | Cloudflare tunnel credentials |
