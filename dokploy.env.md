# Moltbot Dokploy Environment Variables

Copy these variables into your Dokploy service configuration.

## ⚠️ Important: Do NOT Set These Variables

The following variables from `docker-compose.yml` are **NOT** for Dokploy - they are volume mount sources only:

- ❌ `CLAWDBOT_CONFIG_DIR` - **Do not set** (causes volume mount errors)
- ❌ `CLAWDBOT_WORKSPACE_DIR` - **Do not set** (causes volume mount errors)
- ❌ `CLAWDBOT_IMAGE` - Not needed in Dokploy
- ❌ `CLAWDBOT_GATEWAY_PORT` - Use Dokploy's port settings instead
- ❌ `CLAWDBOT_BRIDGE_PORT` - Use Dokploy's port settings instead

Configure volumes through Dokploy's UI, not environment variables.

## Required Variables

```bash
# Setup wizard password (required for first-time /setup access)
SETUP_PASSWORD=

# Gateway authentication token (generate with: openssl rand -hex 32)
CLAWDBOT_GATEWAY_TOKEN=
```

## State Directory (Internal Container Paths)

**IMPORTANT**: Do NOT set `CLAWDBOT_CONFIG_DIR` or `CLAWDBOT_WORKSPACE_DIR` as environment variables in Dokploy.

These are docker-compose volume mount sources, not container env vars. Dokploy handles volumes through its UI, not env vars.

Inside the container, paths are always:
- Config: `/home/node/.clawdbot`
- Workspace: `/home/node/clawd`

Only set the state directory override (for rebrand compatibility):

```bash
# State directory override (internal path, not volume mount)
MOLTBOT_STATE_DIR=/home/node/.clawdbot
CLAWDBOT_STATE_DIR=/home/node/.clawdbot
```

## Gateway Configuration

```bash
# Gateway mode: "local" or "remote"
MOLTBOT_GATEWAY__MODE=local

# Gateway bind address: "local", "lan", or "0.0.0.0"
CLAWDBOT_GATEWAY_BIND=local

# Gateway port (default: 18789)
CLAWDBOT_GATEWAY_PORT=18789
```

## Optional: Claude AI Provider

```bash
# Claude provider credentials (optional - only if using Claude)
CLAUDE_AI_SESSION_KEY=
CLAUDE_WEB_SESSION_KEY=
CLAUDE_WEB_COOKIE=
```

## Optional: Standard Node Variables

```bash
# Node environment (optional)
NODE_ENV=production

# Web server port (optional - may be handled by Dokploy)
PORT=8080
```

---

## Volume Mounts

Configure these persistent volume mounts in Dokploy:

| Container Path | Description |
|----------------|-------------|
| `/home/node/.clawdbot` | Config directory |
| `/home/node/clawd` | Workspace directory |

## Ports to Expose

| Port | Protocol | Description |
|------|----------|-------------|
| 18789 | TCP | Gateway API/Web UI |
| 18790 | TCP | Bridge port (optional) |

## Health Check

```
node dist/index.js health --token "$CLAWDBOT_GATEWAY_TOKEN"
```

## After First Deploy

1. Navigate to `http://<your-dokploy-domain>:18789/setup`
2. Enter your `SETUP_PASSWORD`
3. Complete the setup wizard
4. Configure your AI provider and messaging channels

---

## Traefik Labels (Dokploy)

Add these labels to your Moltbot service in Dokploy for Traefik reverse proxy:

### Basic HTTP Routing

```yaml
# Enable Traefik for this container
traefik.enable=true

# HTTP router (main gateway)
traefik.http.routers.moltbot.rule=Host(`moltbot.your-domain.com`)
traefik.http.routers.moltbot.entrypoints=websecure
traefik.http.routers.moltbot.tls=true
traefik.http.routers.moltbot.tls.certresolver=letsencrypt

# Service configuration
traefik.http.services.moltbot.loadbalancer.server.port=18789

# WebSocket support (required for Gateway)
traefik.http.routers.moltbot.service=moltbot
```

### With HTTPS Redirect (HTTP → HTTPS)

```yaml
# HTTP router (redirects to HTTPS)
traefik.http.routers.moltbot-http.rule=Host(`moltbot.your-domain.com`)
traefik.http.routers.moltbot-http.entrypoints=web
traefik.http.routers.moltbot-http.middlewares=redirect-to-https

# HTTPS redirect middleware
traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true
```

### Setup Wizard Subdomain (Optional)

```yaml
# Separate router for setup wizard
traefik.http.routers.moltbot-setup.rule=Host(`setup.your-domain.com`)
traefik.http.routers.moltbot-setup.entrypoints=websecure
traefik.http.routers.moltbot-setup.tls=true
traefik.http.routers.moltbot-setup.tls.certresolver=letsencrypt
traefik.http.routers.moltbot-setup.service=moltbot
```

### Full Example (All Labels Combined)

```yaml
traefik.enable=true
traefik.http.routers.moltbot.rule=Host(`moltbot.your-domain.com`)
traefik.http.routers.moltbot.entrypoints=websecure
traefik.http.routers.moltbot.tls=true
traefik.http.routers.moltbot.tls.certresolver=letsencrypt
traefik.http.routers.moltbot.service=moltbot
traefik.http.services.moltbot.loadbalancer.server.port=18789
traefik.http.routers.moltbot-http.rule=Host(`moltbot.your-domain.com`)
traefik.http.routers.moltbot-http.entrypoints=web
traefik.http.routers.moltbot-http.middlewares=redirect-to-https
traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true
```

---

## Docker Compose Example (for reference)

```yaml
services:
  moltbot:
    image: ghcr.io/moltbot/moltbot:main
    container_name: moltbot
    restart: unless-stopped

    environment:
      - SETUP_PASSWORD=${SETUP_PASSWORD}
      - CLAWDBOT_GATEWAY_TOKEN=${CLAWDBOT_GATEWAY_TOKEN}
      - MOLTBOT_STATE_DIR=/home/node/.clawdbot
      - CLAWDBOT_STATE_DIR=/home/node/.clawdbot
      - MOLTBOT_GATEWAY__MODE=local
      - CLAWDBOT_GATEWAY_BIND=local
      - CLAWDBOT_GATEWAY_PORT=18789

    volumes:
      - moltbot-config:/home/node/.clawdbot
      - moltbot-workspace:/home/node/clawd

    labels:
      - traefik.enable=true
      - traefik.http.routers.moltbot.rule=Host(`moltbot.your-domain.com`)
      - traefik.http.routers.moltbot.entrypoints=websecure
      - traefik.http.routers.moltbot.tls=true
      - traefik.http.routers.moltbot.tls.certresolver=letsencrypt
      - traefik.http.routers.moltbot.service=moltbot
      - traefik.http.services.moltbot.loadbalancer.server.port=18789

volumes:
  moltbot-config:
  moltbot-workspace:
```

### Notes:
- Replace `your-domain.com` with your actual domain
- Ensure `letsencrypt` cert resolver is configured in your Traefik instance
- The Gateway uses WebSocket connections - Traefik handles this automatically
- Port `18789` is the internal container port, not the exposed host port
