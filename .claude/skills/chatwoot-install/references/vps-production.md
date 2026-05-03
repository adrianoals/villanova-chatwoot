# Chatwoot — VPS Production Deployment (Traefik)

This guide covers deploying Chatwoot on a VPS (Ubuntu 22.04/24.04) with Traefik handling TLS termination and Let's Encrypt automatic certificate issuance.

> Why Traefik instead of Nginx + Certbot:
> - SSL is handled by labels in `docker-compose.yaml` — no separate `nginx.conf` and no Certbot cron to maintain
> - Adding a new service (FastAPI, monitoring, etc.) means adding a service block with labels, not editing reverse proxy config
> - Single source of truth: each service declares its own routing in its own compose file

## Table of Contents

1. [Install Docker](#1-install-docker)
2. [Configure Firewall](#2-configure-firewall)
3. [Disable SSH password auth](#3-disable-ssh-password-auth)
4. [Project layout](#4-project-layout)
5. [Generate secrets](#5-generate-secrets)
6. [Deploy Traefik](#6-deploy-traefik)
7. [Deploy Chatwoot (with Traefik labels)](#7-deploy-chatwoot)
8. [Production .env adjustments](#8-production-env-adjustments)
9. [Backup automation](#9-backup-automation)

---

## 1. Install Docker

```bash
curl -fsSL https://get.docker.com | sh
docker --version
docker compose version

# Shared network used by Traefik + apps
docker network create villanova-net
```

---

## 2. Configure Firewall

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp comment "SSH"
ufw allow 80/tcp comment "HTTP (Traefik)"
ufw allow 443/tcp comment "HTTPS (Traefik)"
ufw --force enable
```

> Do NOT expose service ports (3000, 5432, 6379, 8080) — they stay on the internal Docker network only.

---

## 3. Disable SSH password auth

After confirming key-based login works, force key-only:

```bash
# Add hardening at the TOP of /etc/ssh/sshd_config (sshd: first value wins)
sed -i '1i # ===== hardening =====\nPasswordAuthentication no\nPermitRootLogin prohibit-password\nKbdInteractiveAuthentication no\nInclude /etc/ssh/sshd_config.d/*.conf\n' /etc/ssh/sshd_config

sshd -t && systemctl reload ssh

# Verify
sshd -T | grep -E '(passwordauth|permitroot)'
```

Optional: `apt install fail2ban` for brute-force protection.

---

## 4. Project layout

```
/opt/<project>/
├── traefik/
│   ├── docker-compose.yaml    # Traefik with Let's Encrypt
│   └── letsencrypt/           # acme.json (chmod 700)
├── chatwoot/
│   ├── docker-compose.yaml    # Rails + Sidekiq + Postgres + Redis (Traefik labels)
│   └── .env                   # Production secrets (chmod 600)
└── evolution/                 # Optional: Evolution API (WhatsApp)
    ├── docker-compose.yaml
    └── .env
```

---

## 5. Generate secrets

```bash
mkdir -p /opt/<project>/{traefik,chatwoot}
mkdir -p /opt/<project>/traefik/letsencrypt
chmod 700 /opt/<project>/traefik/letsencrypt

# Generate strong random secrets — never commit, never paste in chat
echo "SECRET_KEY_BASE=$(openssl rand -hex 64)"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)"
echo "REDIS_PASSWORD=$(openssl rand -base64 24 | tr -d '/+=' | head -c 32)"
```

Save these to a password manager.

---

## 6. Deploy Traefik

`/opt/<project>/traefik/docker-compose.yaml`:

```yaml
services:
  traefik:
    image: traefik:v3.6
    container_name: traefik
    restart: unless-stopped
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --providers.docker.network=villanova-net
      - --entrypoints.web.address=:80
      - --entrypoints.web.http.redirections.entrypoint.to=websecure
      - --entrypoints.web.http.redirections.entrypoint.scheme=https
      - --entrypoints.websecure.address=:443
      - --certificatesresolvers.le.acme.httpchallenge=true
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web
      - --certificatesresolvers.le.acme.email=YOUR_EMAIL@example.com
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json
      - --log.level=INFO
      - --accesslog=true
      - --api.dashboard=true
      - --api.insecure=true
    ports:
      - "80:80"
      - "443:443"
      - "127.0.0.1:8080:8080"  # dashboard, accessible only via SSH tunnel
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    networks:
      - villanova-net

networks:
  villanova-net:
    external: true
```

Bring up:

```bash
cd /opt/<project>/traefik
docker compose up -d
docker compose logs -f
```

> **Note:** Traefik v3.6+ is required if your Docker daemon is recent (v28+). Older Traefik versions use a Docker SDK that fails with newer daemons (API version mismatch).

---

## 7. Deploy Chatwoot

`/opt/<project>/chatwoot/docker-compose.yaml`:

```yaml
services:
  base: &base
    image: chatwoot/chatwoot:v4.13.0
    env_file: .env
    volumes:
      - storage_data:/app/storage
    networks:
      - default
      - villanova-net

  rails:
    <<: *base
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    entrypoint: docker/entrypoints/rails.sh
    command: ["bundle", "exec", "rails", "s", "-p", "3000", "-b", "0.0.0.0"]
    restart: always
    labels:
      - traefik.enable=true
      - traefik.docker.network=villanova-net
      - traefik.http.routers.chatwoot.rule=Host(`chat.example.com`)
      - traefik.http.routers.chatwoot.entrypoints=websecure
      - traefik.http.routers.chatwoot.tls.certresolver=le
      - traefik.http.services.chatwoot.loadbalancer.server.port=3000

  sidekiq:
    <<: *base
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - NODE_ENV=production
      - RAILS_ENV=production
      - INSTALLATION_ENV=docker
    command: ["bundle", "exec", "sidekiq", "-C", "config/sidekiq.yml"]
    restart: always

  postgres:
    image: pgvector/pgvector:pg16
    restart: always
    env_file: .env
    environment:
      - POSTGRES_DB=${POSTGRES_DATABASE}
      - POSTGRES_USER=${POSTGRES_USERNAME}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USERNAME}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:alpine
    restart: always
    command: ["sh", "-c", "redis-server --requirepass \"$$REDIS_PASSWORD\""]
    env_file: .env
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD-SHELL", "redis-cli -a \"$$REDIS_PASSWORD\" ping"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  storage_data:
  postgres_data:
  redis_data:

networks:
  villanova-net:
    external: true
```

Initialize the database (one-time):

```bash
cd /opt/<project>/chatwoot
docker compose up -d postgres redis
# wait for healthy
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
docker compose up -d
```

Traefik will auto-discover the service via labels and issue a Let's Encrypt cert on first request.

---

## 8. Production .env adjustments

`/opt/<project>/chatwoot/.env`:

```env
SECRET_KEY_BASE=<openssl rand -hex 64>
FRONTEND_URL=https://chat.example.com
DEFAULT_LOCALE=pt_BR
FORCE_SSL=true
ENABLE_ACCOUNT_SIGNUP=false

POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DATABASE=chatwoot_production
POSTGRES_USERNAME=postgres
POSTGRES_PASSWORD=<generated>

REDIS_PASSWORD=<generated>
REDIS_URL=redis://:<generated>@redis:6379

ACTIVE_STORAGE_SERVICE=local
RAILS_LOG_TO_STDOUT=true
LOG_LEVEL=info
RAILS_MAX_THREADS=5

# SMTP (required for password resets, agent invites)
SMTP_ADDRESS=smtp.example.com
SMTP_PORT=587
SMTP_USERNAME=...
SMTP_PASSWORD=...
SMTP_AUTHENTICATION=login
SMTP_ENABLE_STARTTLS_AUTO=true
MAILER_SENDER_EMAIL=Support <support@example.com>
```

`chmod 600 .env` to restrict access.

> **Calling Chatwoot from another container (e.g., Evolution API):** if `FORCE_SSL=true` in Chatwoot, internal HTTP calls to `http://chatwoot-rails-1:3000` get redirected 301→HTTPS, then fail SSL because the internal hostname has no cert. Solution: configure the calling service to use the public HTTPS URL (`https://chat.example.com`) — Traefik handles the TLS termination.

---

## 9. Backup automation

Daily backup of database + storage:

```bash
cat > /opt/<project>/backup.sh <<'BACKUP'
#!/bin/bash
set -e
BACKUP_DIR=/opt/<project>/backups
RETENTION_DAYS=14
mkdir -p "$BACKUP_DIR"
TS=$(date +%Y%m%d_%H%M%S)

# Chatwoot DB
docker compose -f /opt/<project>/chatwoot/docker-compose.yaml \
  exec -T postgres pg_dump -U postgres chatwoot_production \
  | gzip > "$BACKUP_DIR/chatwoot_$TS.sql.gz"

# Chatwoot storage (uploaded files)
docker run --rm \
  -v <project>_storage_data:/data \
  -v "$BACKUP_DIR":/backup \
  alpine tar czf "/backup/storage_$TS.tar.gz" -C /data .

find "$BACKUP_DIR" -name "*.gz" -mtime +$RETENTION_DAYS -delete
BACKUP

chmod +x /opt/<project>/backup.sh

# Daily at 03:00
(crontab -l 2>/dev/null; echo "0 3 * * * /opt/<project>/backup.sh >> /opt/<project>/backup.log 2>&1") | crontab -
```

Off-site copies: rsync to S3 / Backblaze B2 / another VPS via cron.
