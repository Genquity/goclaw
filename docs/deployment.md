# Deployment

## Platform
Coolify (Self-Hosted) — Your VPS dashboard

## Production URL
`https://goclaw.yourdomain.com` (example subdomain)

## Stack
- **Backend**: Go + Docker (goclaw container)
- **Frontend**: React SPA (embedded in backend, served at same port)
- **Database**: PostgreSQL via Supabase (external)
- **Reverse Proxy/SSL**: Caddy (built into Coolify)
- **DNS**: Cloudflare (subdomain pointing to VPS IP)

---

## Prerequisites

1. VPS with Ubuntu 20.04+ (tested on 22.04/24.04)
2. Domain with Cloudflare (subdomain ready)
3. Supabase PostgreSQL connection string
4. Git repository accessible from VPS

---

## Step 1: Install Coolify on VPS

```bash
# SSH into your VPS
ssh root@your-vps-ip

# Install Coolify
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash

# Follow the on-screen instructions to create admin account
# Access Coolify at https://coolify.yourdomain.com
```

---

## Step 2: Configure Cloudflare DNS

In Cloudflare dashboard for your domain:

1. Go to **DNS** → **Records**
2. Add **A record**:
   - **Name**: `goclaw` (your subdomain)
   - **IPv4 address**: Your VPS IP
   - **Proxy status**: **Proxied** (orange cloud) — enables Cloudflare proxy + Caddy SSL
3. Save

> **Note**: Coolify's Caddy will automatically request SSL certs via Let's Encrypt for proxied Cloudflare domains.

---

## Step 3: Deploy via Coolify

### Option A: From Git Repository (Recommended)

1. In Coolify dashboard, click **Add New Resource** → **Application**
2. Select **GitHub** or **GitLab** and authorize
3. Choose your `goclaw` repository
4. **Build Pack**: Select `Dockerfile` (Coolify auto-detects)
5. **Port / Exposed Port / Container Port**: `18790`
6. **HTTP Port / Service Port**: `18790`
7. If Coolify exposes Docker build arguments separately, set:
   - `ENABLE_EMBEDUI=true`
   - `ENABLE_PYTHON=true`
   - `ENABLE_FULL_SKILLS=false`

### Option B: From Docker Compose

1. Create a new **Docker Compose** resource in Coolify
2. Paste your `docker-compose.yml` content
3. Coolify parses and deploys

---

## Step 4: Configure Build Args

For a **Dockerfile-based** Coolify deployment, these values must be configured as
**build arguments**, not only as runtime environment variables:

| Build Arg | Value | Notes |
|---|---|---|
| `ENABLE_EMBEDUI` | `true` | Required to embed the React SPA into the Go binary |
| `ENABLE_PYTHON` | `true` | Recommended for Python-backed skills |
| `ENABLE_FULL_SKILLS` | `false` | Set `true` only if you want all skill deps preinstalled |
| `ENABLE_OTEL` | `false` | Optional |

`ENABLE_EMBEDUI` is a Docker build arg consumed by the Dockerfile. If you set it
only as a runtime env var, the container will start without embedded frontend assets.

---

## Step 5: Configure Environment Variables

In Coolify → Your App → **Environment Variables**, add:

| Variable | Value | Notes |
|---|---|---|
| `GOCLAW_HOST` | `0.0.0.0` | Bind to all interfaces |
| `GOCLAW_PORT` | `18790` | Container port |
| `GOCLAW_CONFIG` | `/app/data/config.json` | Default, don't change |
| `GOCLAW_POSTGRES_DSN` | `postgres://postgres.[ref]:[pass]@aws-[region].pooler.supabase.com:6543/postgres?sslmode=require` | **Supabase connection string with `sslmode=require`** |
| `GOCLAW_ENCRYPTION_KEY` | `<generate-32-char-key>` | Run: `openssl rand -base64 32` |
| `GOCLAW_GATEWAY_TOKEN` | `<generate-secure-token>` | Run: `openssl rand -hex 32` |
| `GOCLAW_SKILLS_DIR` | `/app/data/skills` | Default, don't change |
| `GOCLAW_ALLOWED_ORIGINS` | `https://goclaw.yourdomain.com` | Optional but recommended for production WebSocket origin checks |

---

## Step 6: Configure Domain in Coolify

1. In Coolify → Your App → **Domains**
2. Add new domain: `goclaw.yourdomain.com`
3. Select **HTTPS** with auto-SSL (Caddy/Let's Encrypt)
4. Save

Coolify will provision SSL certificate automatically.

---

## Step 7: Database Migration

GoClaw runs `goclaw upgrade` automatically on container start when `GOCLAW_POSTGRES_DSN`
is set. On a fresh database this can take more than a minute while schema `0 -> 35`
is applied, so the container may stay in "starting" state for a while before `/health`
responds.

If you prefer to run migrations manually instead of waiting for first boot:

```bash
# Via Coolify terminal (or SSH into server)
docker exec -it goclaw goclaw migrate up

# Or from source if building locally:
go run ./cmd/goclaw migrate up
```

---

## Step 8: Verify Deployment

1. Visit `https://goclaw.yourdomain.com`
2. Complete onboard wizard
3. Create your first agent

Expected behavior:
- The embedded frontend is served from the same origin as the API.
- HTTP requests go to `https://goclaw.yourdomain.com/v1/...`
- WebSocket connects to `wss://goclaw.yourdomain.com/ws`

---

## Environment Variables Reference

### Core (Required)
```bash
GOCLAW_POSTGRES_DSN=postgres://postgres.[project-ref]:[password]@aws-0-[region].pooler.supabase.com:6543/postgres?sslmode=require
GOCLAW_ENCRYPTION_KEY=<32-char-secure-key>
GOCLAW_GATEWAY_TOKEN=<secure-token>
```

### Optional
```bash
GOCLAW_PORT=18790
GOCLAW_HOST=0.0.0.0
GOCLAW_CONFIG=/app/data/config.json
GOCLAW_DATA_DIR=/app/data
GOCLAW_WORKSPACE=/app/workspace
GOCLAW_SKILLS_DIR=/app/data/skills
GOCLAW_ALLOWED_ORIGINS=https://goclaw.yourdomain.com
ENABLE_OTEL=false
GOCLAW_TRACE_VERBOSE=0
```

### Build Args (Dockerfile Deployments)
```bash
ENABLE_EMBEDUI=true
ENABLE_PYTHON=true
ENABLE_FULL_SKILLS=false
ENABLE_OTEL=false
```

---

## Supabase Connection String Format

```
postgres://postgres.[PROJECT_REF]:[PASSWORD]@[HOST]:6543/postgres?sslmode=require
```

- **PROJECT_REF**: Found in Supabase Dashboard → Settings → API
- **PASSWORD**: Your PostgreSQL user password
- **HOST**: From Supabase Dashboard → Settings → Connection Pooling
- **Port**: `6543` (pooler) or `5432` (direct)

---

## Rollback

In Coolify dashboard:
1. Go to Your App → **Deployments**
2. Select a previous working deployment
3. Click **Rollback**

---

## Troubleshooting

### Database Connection Failed
- Verify `GOCLAW_POSTGRES_DSN` has `sslmode=require`
- Check Supabase IP allowlist includes your VPS IP
- Test connection: `psql "<GOCLAW_POSTGRES_DSN>"`

### Container Becomes Unhealthy During First Deploy
- A fresh Postgres database starts at schema `0`, so first boot must apply all migrations before the HTTP server starts.
- Wait for the initial upgrade to finish; the image healthcheck now allows extra startup time for this path.
- If startup still fails, inspect container logs for migration errors or blocked database connections.

### Cloudflare Shows 502
- A Cloudflare `502` usually means Coolify could not get a valid response from the GoClaw container, not that the SPA routing is wrong.
- For Dockerfile deployments, make sure Coolify proxies to internal port `18790`.
- Ensure `GOCLAW_HOST=0.0.0.0` and `GOCLAW_PORT=18790` are set as runtime env vars.
- Ensure `ENABLE_EMBEDUI=true` is configured as a Docker build arg so the frontend is actually embedded into the built image.
- If Coolify keeps rolling back the deployment, inspect the container logs for startup or migration failures before debugging Cloudflare.

### SSL Certificate Issues
- Ensure DNS is proxied through Cloudflare
- Wait 5 minutes for DNS propagation
- Check Coolify logs: `docker logs coolify`

### Container Out of Memory
- Reduce memory limit in Coolify resource settings
- Current default: 1G

---

## Docker Compose (Alternative to Coolify)

If deploying manually without Coolify:

```bash
# Clone repo
git clone https://github.com/yourrepo/goclaw.git
cd goclaw

# Create .env
cat > .env << 'EOF'
GOCLAW_POSTGRES_DSN=postgres://user:pass@host:5432/db?sslmode=require
GOCLAW_ENCRYPTION_KEY=your-32-char-key
GOCLAW_GATEWAY_TOKEN=your-gateway-token
EOF

# Start with Caddy reverse proxy
docker compose -f docker-compose.yml up -d
```

For Caddy reverse proxy with Cloudflare SSL:
```bash
# docker-compose.proxy.yml
services:
  caddy:
    image: caddy:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy-data:/data
    depends_on:
      - goclaw
```
