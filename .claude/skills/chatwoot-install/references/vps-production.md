# Chatwoot — VPS Production Deployment

This guide covers deploying Chatwoot on a VPS (Ubuntu 22.04/24.04) with Nginx and SSL.

## Table of Contents

1. [Install Docker](#1-install-docker)
2. [Configure Firewall](#2-configure-firewall)
3. [Deploy Chatwoot](#3-deploy-chatwoot)
4. [Configure Nginx](#4-configure-nginx)
5. [SSL with Let's Encrypt](#5-ssl-with-lets-encrypt)
6. [Production .env Adjustments](#6-production-env-adjustments)
7. [Systemd Service (auto-start)](#7-systemd-service)
8. [Backup Automation](#8-backup-automation)

---

## 1. Install Docker

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Add current user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt install docker-compose-plugin -y

# Verify
docker --version
docker compose version
```

Log out and back in for the group change to take effect.

---

## 2. Configure Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

> Do NOT expose ports 3000, 5432, or 6379 — they stay on localhost only.

---

## 3. Deploy Chatwoot

```bash
mkdir -p ~/chatwoot && cd ~/chatwoot

# Create docker-compose.yaml and .env as described in SKILL.md
# The files are identical to the local setup

# Generate secrets
openssl rand -hex 64  # for SECRET_KEY_BASE

# Prepare database
docker compose run --rm rails bundle exec rails db:chatwoot_prepare

# Start
docker compose up -d
```

---

## 4. Configure Nginx

```bash
sudo apt install nginx -y
```

Create the site config:

```bash
sudo tee /etc/nginx/sites-available/chatwoot > /dev/null <<'NGINX'
server {
    listen 80;
    server_name chat.yourdomain.com;

    underscores_in_headers on;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 86400;
        proxy_send_timeout 86400;
    }

    client_max_body_size 50M;
}
NGINX
```

Enable and test:

```bash
sudo ln -sf /etc/nginx/sites-available/chatwoot /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

> **Critical:** The `underscores_in_headers on` directive is required because Chatwoot uses `api_access_token` in headers, which contains underscores. Without this, API authentication will silently fail.

---

## 5. SSL with Let's Encrypt

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d chat.yourdomain.com
```

Certbot will automatically modify the Nginx config to add SSL and set up auto-renewal.

Verify auto-renewal:

```bash
sudo certbot renew --dry-run
```

---

## 6. Production .env Adjustments

Update these variables for production:

```env
FRONTEND_URL=https://chat.yourdomain.com
FORCE_SSL=true
ENABLE_ACCOUNT_SIGNUP=false

# Email is important in production for password resets and notifications
SMTP_ADDRESS=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_AUTHENTICATION=login
SMTP_ENABLE_STARTTLS_AUTO=true
MAILER_SENDER_EMAIL=Chatwoot <your@gmail.com>
```

After changing `.env`, restart:

```bash
docker compose down && docker compose up -d
```

---

## 7. Systemd Service

Create a systemd service so Chatwoot starts automatically on boot:

```bash
sudo tee /etc/systemd/system/chatwoot.service > /dev/null <<'SERVICE'
[Unit]
Description=Chatwoot Docker Compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/YOUR_USER/chatwoot
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
SERVICE
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable chatwoot
```

---

## 8. Backup Automation

Create a daily backup script:

```bash
sudo tee /home/YOUR_USER/chatwoot/backup.sh > /dev/null <<'BACKUP'
#!/bin/bash
BACKUP_DIR="/home/YOUR_USER/chatwoot/backups"
RETENTION_DAYS=7

mkdir -p "$BACKUP_DIR"

# Dump database
docker compose -f /home/YOUR_USER/chatwoot/docker-compose.yaml \
  exec -T postgres pg_dump -U postgres chatwoot_production \
  | gzip > "$BACKUP_DIR/chatwoot_$(date +%Y%m%d_%H%M%S).sql.gz"

# Remove old backups
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "[$(date)] Backup completed"
BACKUP

chmod +x /home/YOUR_USER/chatwoot/backup.sh
```

Add to crontab (runs daily at 2:00 AM):

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * /home/YOUR_USER/chatwoot/backup.sh >> /home/YOUR_USER/chatwoot/backups/backup.log 2>&1") | crontab -
```
