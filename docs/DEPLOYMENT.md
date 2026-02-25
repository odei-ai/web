# Deployment Guide

This document describes the deployment workflow for the ODEI web platform at `api.odei.ai`.

---

## Overview

Deployments use an **atomic symlink swap** to achieve zero-downtime releases. The process:

1. Export fresh data (Sentinel feed, world model projection)
2. Validate required files and social metadata
3. Upload to a staging directory on the server via tarball over SSH
4. Run an atomic release script that swaps the symlink
5. Verify the public endpoint

---

## Prerequisites

### Local Machine

- **Node.js 20+** -- for running export scripts
- **SSH access** to the production server via alias `google-cloud-api`
  - Key: `~/.ssh/id_ed25519_gcloud3`
  - User: `ai`
  - Host: `34.18.183.162`
- **Neo4j access** -- the export scripts read from the production graph

### Production Server

- **nginx** -- serves static files and proxies `/api/*` to port 8800
- **systemd service** `odei-api.service` -- manages the Node.js API server
- **Directory structure:**
  ```
  /srv/api.odei.ai/
  ├── current -> /srv/api.odei.ai/releases/20260225-120000   (symlink)
  ├── staging/              (upload target, wiped each deploy)
  ├── releases/             (timestamped release directories)
  └── data/                 (persistent data: capsules, intakes, context)
  ```

**Important:** The API server is managed by **systemd**, NOT PM2. Never add it to PM2 -- this causes `EADDRINUSE` crash loops.

```bash
# Restart API server
sudo systemctl restart odei-api

# Check status
sudo systemctl status odei-api

# View logs
sudo journalctl -u odei-api -f
```

---

## Deployment Steps

### Automated (Recommended)

Run the deploy script from the ODEI root:

```bash
./scripts/deploy-api-site.sh
```

Or specify a custom source directory:

```bash
./scripts/deploy-api-site.sh /path/to/api-site
```

The script performs these steps:

#### Step 0/4: Export Sentinel Feed

```bash
node scripts/export-sentinel-feed.js deploy/api-site/sentinel-feed.json
```

Exports the latest Sentinel feed data to `sentinel-feed.json`.

#### Step 1/4: Export World Model Projection

```bash
node scripts/export-world-model-demo.js deploy/api-site/world-model-demo.json
```

Exports the public world model graph snapshot. If the export fails, the existing file is preserved (the deploy continues).

#### Step 1.5/4: Validate Social Metadata

The script checks `index.html` and `worldmodel.html` for:

- No legacy social copy (e.g., "$ODAI on Base" text)
- Required Open Graph tags: `og:title`, `og:description`
- Required Twitter Card tags: `twitter:title`, `twitter:description`

Validation failure aborts the deploy.

#### Step 2/4: Upload

```bash
tar -C deploy/api-site -cf - . | ssh google-cloud-api "mkdir -p /srv/api.odei.ai/staging; \
  find /srv/api.odei.ai/staging -mindepth 1 -maxdepth 1 -exec rm -rf {} +; \
  tar -C /srv/api.odei.ai/staging -xf -"
```

Creates a tarball of the local `deploy/api-site/` directory (excluding `.DS_Store`), pipes it over SSH, and extracts into the staging directory. The staging directory is cleaned before extraction.

#### Step 3/4: Atomic Release

```bash
ssh google-cloud-api "/usr/local/bin/api-site-deploy /srv/api.odei.ai/staging"
```

The server-side `api-site-deploy` script:

1. Creates a timestamped release directory (e.g., `/srv/api.odei.ai/releases/20260225-120000`)
2. Copies staging contents into it
3. Atomically swaps the `current` symlink using `ln -sfn`
4. Optionally prunes old releases (keeps last N)

Because nginx serves from the `current` symlink, the switch is instantaneous with zero downtime.

#### Step 4/4: Verify

```bash
curl -sS -I -m 15 https://api.odei.ai/
```

Checks that the public endpoint returns a successful HTTP response.

---

## Manual Deployment

If the automated script is unavailable, deploy manually:

```bash
# 1. Export data
node scripts/export-sentinel-feed.js deploy/api-site/sentinel-feed.json
node scripts/export-world-model-demo.js deploy/api-site/world-model-demo.json

# 2. Upload
scp -r deploy/api-site/* google-cloud-api:/srv/api.odei.ai/staging/

# 3. SSH into server and swap
ssh google-cloud-api
sudo /usr/local/bin/api-site-deploy /srv/api.odei.ai/staging

# 4. Verify
curl -I https://api.odei.ai/
```

---

## Post-Deploy Verification

After deployment, verify these endpoints:

```bash
# Health check
curl -s https://api.odei.ai/health | jq .ok

# API v2 health (checks upstream connectivity)
curl -s https://api.odei.ai/api/v2/health | jq '.data.status'

# World model (should have nodes)
curl -s https://api.odei.ai/api/worldmodel/live | jq '.nodeCount'

# Token price (should return live data)
curl -s https://api.odei.ai/api/token/price | jq '.data.price'

# OpenAPI spec
curl -s https://api.odei.ai/openapi.json | jq '.info.title'

# Static pages (check for 200)
for path in "/" "/token/" "/economics/" "/integrate/" "/docs/"; do
  status=$(curl -s -o /dev/null -w "%{http_code}" "https://api.odei.ai${path}")
  echo "${path} -> ${status}"
done
```

---

## Rollback

To roll back to a previous release:

```bash
ssh google-cloud-api

# List available releases
ls -lt /srv/api.odei.ai/releases/ | head -5

# Swap symlink to previous release
sudo ln -sfn /srv/api.odei.ai/releases/PREVIOUS_TIMESTAMP /srv/api.odei.ai/current

# Verify
curl -I https://api.odei.ai/
```

No server restart is needed -- nginx picks up the new symlink target immediately for static files.

If the API server code changed, restart it:

```bash
sudo systemctl restart odei-api
```

---

## Environment Variables

These are configured in the systemd service file (`/etc/systemd/system/odei-api.service`):

| Variable | Value | Description |
|----------|-------|-------------|
| `ODEI_API_HOST` | `127.0.0.1` | Bind to localhost only (nginx is the public boundary) |
| `ODEI_API_PORT` | `8800` | API server port |
| `ODEI_API_DATA_DIR` | `/srv/api.odei.ai/data` | Persistent data directory |
| `ODEI_API_GRAPH_FALLBACK_PATH` | `/srv/api.odei.ai/current/world-model-demo.json` | Fallback graph snapshot |
| `ODEI_TELEGRAM_CHAT_ID` | (configured) | Telegram chat for alerts |
| `CLAWGIG_WEBHOOK_SECRET` | (configured) | ClawGig webhook authentication |

---

## Nginx Configuration

nginx serves two roles:

1. **Static files** from `/srv/api.odei.ai/current/` for all non-API paths
2. **Reverse proxy** to `127.0.0.1:8800` for `/api/*`, `/health`, `/openapi.json`, `/blink/*`, `/fetch/*`

Key nginx directives:

```nginx
server {
    listen 443 ssl http2;
    server_name api.odei.ai;

    root /srv/api.odei.ai/current;
    index index.html;

    # Static files
    location / {
        try_files $uri $uri/ $uri/index.html =404;
    }

    # API proxy
    location /api/ {
        proxy_pass http://127.0.0.1:8800;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

    location /health {
        proxy_pass http://127.0.0.1:8800;
    }

    location /openapi.json {
        proxy_pass http://127.0.0.1:8800;
    }

    location /blink/ {
        proxy_pass http://127.0.0.1:8800;
    }

    location /fetch/ {
        proxy_pass http://127.0.0.1:8800;
    }
}
```

---

## SSH Configuration

The deploy script uses these SSH options for reliability:

| Option | Value | Purpose |
|--------|-------|---------|
| `ConnectTimeout` | 20s | Fail fast on unreachable server |
| `ServerAliveInterval` | 15s | Detect dead connections |
| `ServerAliveCountMax` | 2 | Give up after 2 missed keepalives |
| `StrictHostKeyChecking` | `accept-new` | Accept new keys, reject changed ones |
| `BatchMode` | yes | No interactive prompts (CI-safe) |

**Note:** SSH commands to this server can sometimes execute twice. For critical operations, use heredoc syntax:

```bash
ssh -T google-cloud-api << 'EOF'
# your commands here
EOF
```

---

## Troubleshooting

### EADDRINUSE on port 8800

The API server is in both systemd and PM2. Remove from PM2:

```bash
ssh google-cloud-api "pm2 delete api-server && pm2 save"
sudo systemctl restart odei-api
```

### Deploy fails at social metadata validation

Check `index.html` and `worldmodel.html` for missing or legacy meta tags. Required:

```html
<meta property="og:title" content="..." />
<meta property="og:description" content="..." />
<meta name="twitter:title" content="..." />
<meta name="twitter:description" content="..." />
```

### World model export fails

The export reads from Neo4j. If it fails:

1. Check Neo4j connectivity: `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`
2. The deploy continues with the existing `world-model-demo.json` (non-fatal)

### nginx returns 502

The API server is down. Restart:

```bash
ssh google-cloud-api "sudo systemctl restart odei-api && sudo systemctl status odei-api"
```
