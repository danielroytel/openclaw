# OpenClaw Dokploy Environment Variables

Copy these variables into your Dokploy service configuration.

## ⚠️ Important: Do NOT Set These Variables

The following variables are **NOT** for Dokploy - they are volume mount sources only:

- ❌ `OPENCLAW_CONFIG_DIR` - **Do not set** (causes volume mount errors)
- ❌ `OPENCLAW_WORKSPACE_DIR` - **Do not set** (causes volume mount errors)
- ❌ `CLAWDBOT_CONFIG_DIR` - Legacy name, do not set
- ❌ `CLAWDBOT_WORKSPACE_DIR` - Legacy name, do not set

Configure volumes through Dokploy's UI, not environment variables.

## Image Variable (Optional)

```bash
# Override the Docker image (defaults to openclaw:local)
OPENCLAW_IMAGE=openclaw:local
```

## Required Variables

```bash
# Setup wizard password (required for first-time /setup access)
SETUP_PASSWORD=

# Gateway authentication token (generate with: openssl rand -hex 32)
OPENCLAW_GATEWAY_TOKEN=
```

## Optional: Ollama Provider (Local LLMs)

```bash
# Enable Ollama for local LLMs (any value works; Ollama doesn't require a real key)
OLLAMA_API_KEY=ollama-local
```

> **Note**: For Ollama on a different host, you'll need to configure the custom URL via the app's **Configuration → Models** page. See https://docs.openclaw.ai/providers/ollama

## Optional: Doppler Secrets

```bash
# Doppler API key (for manual configuration - requires CLI setup in container)
DOPPLER_API_KEY=dplt_xxxxx
```

The Doppler CLI is **not** pre-installed in this configuration. To use Doppler:

1. Install the CLI manually in the container:
   ```bash
   docker exec -it openclaw-gateway sh -c "curl -Ls https://cli.doppler.com/install.sh | sh"
   ```

2. Configure and use:
   ```bash
   docker exec -it openclaw-gateway doppler configure
   docker exec openclaw-gateway doppler run -- openclaw channels status
   ```

## State Directory (Internal Container Paths)

**IMPORTANT**: Do NOT set `OPENCLAW_CONFIG_DIR` or `OPENCLAW_WORKSPACE_DIR` as environment variables in Dokploy.

These are docker-compose volume mount sources, not container env vars. Dokploy handles volumes through its UI, not env vars.

Inside the container, paths are always:
- Config: `/home/node/.openclaw` (legacy: `/home/node/.clawdbot`)
- Workspace: `/home/node/openclaw` (legacy: `/home/node/clawd`)

State directory override (for custom paths):
```bash
# State directory override (internal path, not volume mount)
OPENCLAW_STATE_DIR=/home/node/.openclaw

# Legacy fallback (still supported)
CLAWDBOT_STATE_DIR=/home/node/.clawdbot
MOLTBOT_STATE_DIR=/home/node/.clawdbot
```

## Gateway Configuration

```bash
# Gateway mode: "local" or "remote"
OPENCLAW_GATEWAY__MODE=local

# Gateway bind address: "loopback", "lan", "tailnet", "auto", or "custom"
OPENCLAW_GATEWAY_BIND=lan

# Gateway port (default: 18789)
OPENCLAW_GATEWAY_PORT=18789

# Bridge port (default: 18790)
OPENCLAW_BRIDGE_PORT=18790
```

## Optional: Control UI Origins

```bash
# Allowed origins for Control UI WebSocket/CORS (for Tailscale Funnel domains)
OPENCLAW_GATEWAY_CONTROL_UI_ALLOWED_ORIGINS=https://openclaw.tailca039.ts.net,https://openclaw.tailca039.ts.net:18789
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

## Legacy Environment Variables (Still Supported)

For backward compatibility, legacy variable names are still supported:

| Legacy Name | New Name |
|------------|----------|
| `CLAWDBOT_GATEWAY_TOKEN` | `OPENCLAW_GATEWAY_TOKEN` |
| `CLAWDBOT_GATEWAY_BIND` | `OPENCLAW_GATEWAY_BIND` |
| `MOLTBOT_GATEWAY__MODE` | `OPENCLAW_GATEWAY__MODE` |
| `CLAWDBOT_STATE_DIR` | `OPENCLAW_STATE_DIR` |

---

## Volume Mounts

Configure these persistent volume mounts in Dokploy:

| Container Path | Description |
|----------------|-------------|
| `/home/node/.openclaw` | Config directory (legacy fallback: `/home/node/.clawdbot`) |
| `/home/node/openclaw` | Workspace directory (legacy fallback: `/home/node/clawd`) |

**Note**: The compose file includes both new (`.openclaw`) and legacy (`.clawdbot`) volume mounts for backward compatibility with existing data.

## Ports to Expose

| Port | Protocol | Description |
|------|----------|-------------|
| 18789 | TCP | Gateway API/Web UI |
| 18790 | TCP | Bridge port (optional) |

## Health Check

```
node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

## After First Deploy

1. Navigate to `http://<your-dokploy-domain>:18789/setup`
2. Enter your `SETUP_PASSWORD`
3. Complete the setup wizard
4. Configure your AI provider and messaging channels

---

## Traefik Labels (Dokploy)

Add these labels to your OpenClaw service in Dokploy for Traefik reverse proxy:

### Basic HTTP Routing

```yaml
# Enable Traefik for this container
traefik.enable=true

# HTTP router (main gateway)
traefik.http.routers.openclaw.rule=Host(`openclaw.your-domain.com`)
traefik.http.routers.openclaw.entrypoints=websecure
traefik.http.routers.openclaw.tls=true
traefik.http.routers.openclaw.tls.certresolver=letsencrypt

# Service configuration
traefik.http.services.openclaw.loadbalancer.server.port=18789

# WebSocket support (required for Gateway)
traefik.http.routers.openclaw.service=openclaw
```

### With HTTPS Redirect (HTTP → HTTPS)

```yaml
# HTTP router (redirects to HTTPS)
traefik.http.routers.openclaw-http.rule=Host(`openclaw.your-domain.com`)
traefik.http.routers.openclaw-http.entrypoints=web
traefik.http.routers.openclaw-http.middlewares=redirect-to-https

# HTTPS redirect middleware
traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true
```

### Full Example (All Labels Combined)

```yaml
traefik.enable=true
traefik.http.routers.openclaw.rule=Host(`openclaw.your-domain.com`)
traefik.http.routers.openclaw.entrypoints=websecure
traefik.http.routers.openclaw.tls=true
traefik.http.routers.openclaw.tls.certresolver=letsencrypt
traefik.http.routers.openclaw.service=openclaw
traefik.http.services.openclaw.loadbalancer.server.port=18789
traefik.http.routers.openclaw-http.rule=Host(`openclaw.your-domain.com`)
traefik.http.routers.openclaw-http.entrypoints=web
traefik.http.routers.openclaw-http.middlewares=redirect-to-https
traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
traefik.http.middlewares.redirect-to-https.redirectscheme.permanent=true
```

---

## Docker Compose Example (for reference)

```yaml
services:
  openclaw:
    image: openclaw:local
    container_name: openclaw
    restart: unless-stopped
    init: true

    environment:
      - SETUP_PASSWORD=${SETUP_PASSWORD}
      - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
      - OPENCLAW_STATE_DIR=/home/node/.openclaw
      - OPENCLAW_GATEWAY__MODE=local
      - OPENCLAW_GATEWAY_BIND=lan
      - OLLAMA_API_KEY=ollama-local

    volumes:
      - openclaw-config:/home/node/.openclaw
      - openclaw-workspace:/home/node/openclaw

    labels:
      - traefik.enable=true
      - traefik.http.routers.openclaw.rule=Host(`openclaw.your-domain.com`)
      - traefik.http.routers.openclaw.entrypoints=websecure
      - traefik.http.routers.openclaw.tls=true
      - traefik.http.routers.openclaw.tls.certresolver=letsencrypt
      - traefik.http.routers.openclaw.service=openclaw
      - traefik.http.services.openclaw.loadbalancer.server.port=18789

volumes:
  openclaw-config:
  openclaw-workspace:
```

### Notes:
- Replace `your-domain.com` with your actual domain
- Ensure `letsencrypt` cert resolver is configured in your Traefik instance
- The Gateway uses WebSocket connections - Traefik handles this automatically
- Port `18789` is the internal container port, not the exposed host port
- Both new (`OPENCLAW_*`) and legacy (`CLAWDBOT_*`, `MOLTBOT_*`) environment variables are supported
