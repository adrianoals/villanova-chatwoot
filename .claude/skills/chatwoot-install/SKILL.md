---
name: chatwoot-install
description: >
  Install and configure Chatwoot self-hosted via Docker Compose (with PostgreSQL and Redis).
  Use this skill whenever the user wants to install Chatwoot, set up Chatwoot with Docker,
  deploy Chatwoot on a VPS, configure Chatwoot environment variables, prepare the Chatwoot
  database, set up Nginx/SSL for Chatwoot, or troubleshoot a Chatwoot Docker installation.
  Also use when the user mentions "subir o Chatwoot", "instalar Chatwoot", "configurar Chatwoot",
  or any Docker-based Chatwoot deployment task.
---

# Chatwoot Self-Hosted Installation

This skill guides the complete installation of Chatwoot via Docker Compose, covering local testing and VPS production environments.

## Stack Components

| Service | Image | Purpose |
|---------|-------|---------|
| **Chatwoot (Rails)** | `chatwoot/chatwoot:latest` | Main application server |
| **Chatwoot (Sidekiq)** | `chatwoot/chatwoot:latest` | Background job processor |
| **PostgreSQL** | `pgvector/pgvector:pg16` | Database (conversations, contacts, configs) |
| **Redis** | `redis:alpine` | Cache, queues, real-time features |

---

## Quick Start — Local (Mac/Linux)

### Prerequisites

- Docker Desktop installed and running
- Ports 3000, 5432, 6379 available

### Step 1: Create project directory

```bash
mkdir -p ~/chatwoot && cd ~/chatwoot
```

### Step 2: Generate the docker-compose.yaml

Write the following `docker-compose.yaml`:

```yaml
version: '3'

services:
  base: &base
    image: chatwoot/chatwoot:latest
    env_file: .env
    volumes:
      - storage_data:/app/storage

  rails:
    <<: *base
    depends_on:
      - postgres
      - redis
    ports:
      - '127.0.0.1:3000:3000'
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    entrypoint: docker/entrypoints/rails.sh
    command: ['bundle', 'exec', 'rails', 's', '-p', '3000', '-b', '0.0.0.0']
    restart: always

  sidekiq:
    <<: *base
    depends_on:
      - postgres
      - redis
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ['bundle', 'exec', 'sidekiq', '-C', 'config/sidekiq.yml']
    restart: always

  postgres:
    image: pgvector/pgvector:pg16
    restart: always
    ports:
      - '127.0.0.1:5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=chatwoot_production
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=CHANGE_ME_POSTGRES_PASSWORD
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
    restart: always
    command: ["sh", "-c", "redis-server --requirepass \"$REDIS_PASSWORD\""]
    env_file: .env
    volumes:
      - redis_data:/data
    ports:
      - '127.0.0.1:6379:6379'
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "$$REDIS_PASSWORD", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  storage_data:
  postgres_data:
  redis_data:
```

### Step 3: Generate the .env file

Write the `.env` file with the essential variables. Generate a SECRET_KEY_BASE with:

```bash
openssl rand -hex 64
```

Essential `.env` contents:

```env
# ===== Core =====
SECRET_KEY_BASE=<generated-hex-64>
FRONTEND_URL=http://localhost:3000
DEFAULT_LOCALE=pt_BR
ENABLE_ACCOUNT_SIGNUP=true

# ===== Database =====
POSTGRES_HOST=postgres
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=CHANGE_ME_POSTGRES_PASSWORD
POSTGRES_DATABASE=chatwoot_production
RAILS_ENV=production
RAILS_MAX_THREADS=5

# ===== Redis =====
REDIS_URL=redis://redis:6379
REDIS_PASSWORD=CHANGE_ME_REDIS_PASSWORD

# ===== Email (optional for testing) =====
# SMTP_ADDRESS=smtp.gmail.com
# SMTP_PORT=587
# SMTP_USERNAME=your@gmail.com
# SMTP_PASSWORD=app-password
# SMTP_AUTHENTICATION=login
# SMTP_ENABLE_STARTTLS_AUTO=true
# MAILER_SENDER_EMAIL=Chatwoot <your@gmail.com>

# ===== Logging =====
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=info
```

> **Important:** The `POSTGRES_PASSWORD` in `.env` must match the one in `docker-compose.yaml` under the postgres service.

### Step 4: Prepare the database

```bash
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
```

This creates all tables, runs migrations, and seeds initial data. It may take 1-2 minutes.

### Step 5: Start all services

```bash
docker compose up -d
```

### Step 6: Verify

```bash
# Check all containers are running
docker compose ps

# Test API response (should return HTTP 200)
curl -I http://localhost:3000/api
```

Access `http://localhost:3000` in the browser to create the admin account.

---

## Production — VPS (Ubuntu)

For VPS deployment, read the reference file `references/vps-production.md` which covers:

- Docker installation on Ubuntu
- Nginx reverse proxy configuration (with `underscores_in_headers on`)
- SSL/TLS via Let's Encrypt (Certbot)
- Firewall (UFW) configuration
- Production `.env` adjustments (`FRONTEND_URL`, `FORCE_SSL`, `ENABLE_ACCOUNT_SIGNUP`)

The docker-compose.yaml is the **same** — only the `.env` and reverse proxy differ.

---

## Post-Install Configuration

After Chatwoot is running, configure it for multi-department use:

### Create Admin Account

On first access (`http://localhost:3000`), create the Super Admin account through the web interface.

### API Configuration

Get your API token from **Profile Settings** in the Chatwoot dashboard, then set:

```bash
export CHATWOOT_BASE_URL="http://localhost:3000"
export CHATWOOT_API_TOKEN="your-api-access-token"
export CHATWOOT_ACCOUNT_ID="1"
```

### Create Inboxes (one per department)

```bash
# Use the Chatwoot web dashboard:
# Settings → Inboxes → Add Inbox → Select channel type
# Repeat for each department (Comercial, Suporte, Financeiro, etc.)
```

### Create Teams

```bash
# Settings → Teams → Create New Team
# Assign agents to each team
# Associate teams with inboxes
```

### Configure Agents and Permissions

| Role | Access |
|------|--------|
| Super Admin | Everything — all departments, settings, reports |
| Administrator | Their department(s) — conversations, reports, team management |
| Agent | Own conversations — respond, tag, transfer, internal notes |

### Automation Rules

```bash
# Settings → Automation
# Example rules:
# - Auto-assign conversations by inbox to specific teams
# - Send welcome message on new conversation
# - Add labels based on message content
```

---

## Common Operations

### View logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f rails
docker compose logs -f sidekiq
docker compose logs -f postgres
```

### Restart services

```bash
docker compose restart          # All
docker compose restart rails    # Just Rails
docker compose restart sidekiq  # Just Sidekiq
```

### Update Chatwoot

```bash
docker compose pull
docker compose down
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
docker compose up -d
```

### Backup database

```bash
# Dump
docker compose exec postgres pg_dump -U postgres chatwoot_production > backup_$(date +%Y%m%d_%H%M%S).sql

# Restore
docker compose exec -T postgres psql -U postgres chatwoot_production < backup_file.sql
```

### Reset admin password

```bash
docker compose run --rm rails bundle exec rails console
# Then in console:
# user = User.find_by(email: 'admin@example.com')
# user.update!(password: 'new_password')
```

---

## Troubleshooting

Read `references/troubleshooting.md` for detailed solutions. Quick reference:

| Problem | Likely Cause | Quick Fix |
|---------|-------------|-----------|
| Port 3000 already in use | Another service on that port | `lsof -i :3000` then stop it, or change port in docker-compose |
| Port 6379 already in use | Local Redis (Homebrew) running | `brew services stop redis` then retry |
| Port 5432 already in use | Local PostgreSQL running | `brew services stop postgresql` or change port in docker-compose |
| Database migration fails | Container didn't finish initializing | Wait for postgres healthcheck, then re-run `db:chatwoot_prepare` |
| Redis connection refused | Wrong password or URL | Verify `REDIS_PASSWORD` matches in `.env` and docker-compose |
| `db:chatwoot_prepare` hangs | Postgres not ready | Run `docker compose up -d postgres redis` first, wait 30s, then run prepare |
| Sidekiq not processing jobs | Redis connection issue | Check `docker compose logs sidekiq` for connection errors |
| 502 Bad Gateway (Nginx) | Rails not running on port 3000 | Check `docker compose ps` — rails must be healthy |
| Assets not loading | Missing `FRONTEND_URL` | Set correct URL in `.env` and restart |

---

## Custom Branding (Logo, Name, Favicon)

Chatwoot Community Edition does not have a UI for branding. Use the **Rails runner** method below.

### Change Logo and Platform Name

The `installation_configs` table stores branding as `ActiveSupport::HashWithIndifferentAccess`. Do NOT use raw SQL — it will fail with JSON type errors. Always use Rails runner:

```bash
docker compose exec -T rails bundle exec rails runner '
  InstallationConfig.find_by(name: "INSTALLATION_NAME").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "Company Name")
  )
  InstallationConfig.find_by(name: "BRAND_NAME").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "Company Name")
  )
  InstallationConfig.find_by(name: "LOGO").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "https://example.com/logo.png")
  )
  InstallationConfig.find_by(name: "LOGO_THUMBNAIL").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "https://example.com/logo.png")
  )
  puts "Done!"
'
```

> The logo URL must be publicly accessible (not a local file path).

After updating, flush the Redis cache and restart:

```bash
docker compose exec redis redis-cli -a YOUR_REDIS_PASSWORD FLUSHALL
docker compose restart rails sidekiq
```

Then hard-refresh the browser (Ctrl+Shift+R).

### Available branding configs

| Config Name | What it changes |
|-------------|----------------|
| `INSTALLATION_NAME` | Platform name (shown in titles, emails) |
| `BRAND_NAME` | Brand name (shown in UI) |
| `LOGO` | Main logo (login page, sidebar) |
| `LOGO_THUMBNAIL` | Small logo (favicon area, notifications) |

### Change Favicon

Favicons are static files inside the container at `/app/public/` and `/app/public/packs/`. The required files are:

```
favicon-16x16.png
favicon-32x32.png
favicon-96x96.png
favicon-512x512.png
favicon-badge-16x16.png
favicon-badge-32x32.png
favicon-badge-96x96.png
favicon-badge-512x512.png
```

**Method 1 — Copy into container** (quick, but lost on container recreation):

```bash
# Generate favicons from a source image (Mac)
for size in 16 32 96 512; do
  sips -z $size $size -s format png source.jpg --out "favicon-${size}x${size}.png"
  cp "favicon-${size}x${size}.png" "favicon-badge-${size}x${size}.png"
done

# Copy into container
for f in favicon-*.png; do
  docker compose cp "$f" "rails:/app/public/$f"
  docker compose cp "$f" "rails:/app/public/packs/$f"
done
```

**Method 2 — Volume mount** (persistent, recommended for production):

Add to `docker-compose.yaml` under the `base` service volumes:

```yaml
base: &base
  image: chatwoot/chatwoot:latest
  env_file: .env
  volumes:
    - storage_data:/app/storage
    - ./favicons/favicon-16x16.png:/app/public/favicon-16x16.png
    - ./favicons/favicon-32x32.png:/app/public/favicon-32x32.png
    - ./favicons/favicon-96x96.png:/app/public/favicon-96x96.png
    - ./favicons/favicon-512x512.png:/app/public/favicon-512x512.png
    - ./favicons/favicon-badge-16x16.png:/app/public/favicon-badge-16x16.png
    - ./favicons/favicon-badge-32x32.png:/app/public/favicon-badge-32x32.png
    - ./favicons/favicon-badge-96x96.png:/app/public/favicon-badge-96x96.png
    - ./favicons/favicon-badge-512x512.png:/app/public/favicon-badge-512x512.png
```

With volume mounts, favicons survive container recreation and updates.

---

## Environment Variables Reference

See the full list in `references/env-variables.md`. The essential ones for a working installation are:

| Variable | Required | Description |
|----------|----------|-------------|
| `SECRET_KEY_BASE` | Yes | 64-char hex — cookie signing |
| `FRONTEND_URL` | Yes | Public URL of the application |
| `POSTGRES_HOST` | Yes | Database host (use `postgres` for Docker) |
| `POSTGRES_PASSWORD` | Yes | Database password |
| `REDIS_URL` | Yes | Redis connection URL |
| `REDIS_PASSWORD` | Yes | Redis password |
| `DEFAULT_LOCALE` | No | Default language (`pt_BR` for Portuguese) |
| `ENABLE_ACCOUNT_SIGNUP` | No | Allow new signups (set `false` in production) |
