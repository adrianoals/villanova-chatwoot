# Chatwoot — Troubleshooting Guide

## Table of Contents

1. [Installation Issues](#1-installation-issues)
2. [Database Issues](#2-database-issues)
3. [Redis Issues](#3-redis-issues)
4. [Application Issues](#4-application-issues)
5. [Nginx / SSL Issues](#5-nginx--ssl-issues)
6. [Performance Issues](#6-performance-issues)

---

## 1. Installation Issues

### Port 3000 already in use

```bash
# Find what's using port 3000
lsof -i :3000

# Option A: Stop the other process
kill -9 <PID>

# Option B: Change Chatwoot port in docker-compose.yaml
# Change '127.0.0.1:3000:3000' to '127.0.0.1:3001:3000'
# Then update FRONTEND_URL in .env to use the new port
```

### Port 5432 (PostgreSQL) conflict

If you already have PostgreSQL installed locally:

```bash
# Check
sudo systemctl status postgresql

# Stop local PostgreSQL
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Or change the mapped port in docker-compose.yaml
# Change '127.0.0.1:5432:5432' to '127.0.0.1:5433:5432'
```

### Docker permission denied

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Apply without logout (current session only)
newgrp docker
```

### Docker Compose version error

If you see errors about `version` field in docker-compose.yaml:

```bash
# Check version
docker compose version

# If using old docker-compose (v1), upgrade:
sudo apt remove docker-compose
sudo apt install docker-compose-plugin
```

---

## 2. Database Issues

### db:chatwoot_prepare fails or hangs

This usually means PostgreSQL isn't ready yet:

```bash
# Step 1: Start only postgres and redis first
docker compose up -d postgres redis

# Step 2: Wait for postgres to be healthy
docker compose ps  # check STATUS column shows "healthy"

# Step 3: Then run prepare
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
```

### Database migration error after update

```bash
# Check current migration status
docker compose run --rm rails bundle exec rails db:migrate:status

# Force run migrations
docker compose run --rm rails bundle exec rails db:migrate

# If still failing, check logs
docker compose logs postgres
```

### "relation does not exist" error

The database wasn't properly initialized:

```bash
docker compose down
docker volume rm chatwoot_postgres_data  # WARNING: destroys all data
docker compose up -d postgres redis
sleep 10
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
docker compose up -d
```

### Cannot connect to database

```bash
# Verify postgres is running
docker compose ps postgres

# Check credentials match between .env and docker-compose.yaml
grep POSTGRES_PASSWORD .env
grep POSTGRES_PASSWORD docker-compose.yaml
# These MUST be identical

# Test connection from rails container
docker compose run --rm rails bundle exec rails dbconsole
```

---

## 3. Redis Issues

### Redis connection refused

```bash
# Check redis is running
docker compose ps redis

# Verify password
docker compose exec redis redis-cli -a YOUR_REDIS_PASSWORD ping
# Should return: PONG

# Check REDIS_URL format in .env
# Correct: redis://redis:6379
# Wrong:   redis://localhost:6379 (inside Docker, use service name)
```

### Redis NOAUTH error

The password in `.env` doesn't match what Redis started with:

```bash
# Restart redis with correct password
docker compose restart redis

# If that doesn't work, recreate
docker compose down redis
docker compose up -d redis
```

---

## 4. Application Issues

### Blank page / assets not loading

```bash
# Check FRONTEND_URL is correct
grep FRONTEND_URL .env

# Local: FRONTEND_URL=http://localhost:3000
# VPS:   FRONTEND_URL=https://chat.yourdomain.com

# Precompile assets
docker compose run --rm rails bundle exec rails assets:precompile

# Restart
docker compose restart rails
```

### Sidekiq not processing jobs

```bash
# Check sidekiq logs
docker compose logs -f sidekiq

# Common: Redis connection issue
# Verify REDIS_URL and REDIS_PASSWORD in .env

# Restart sidekiq
docker compose restart sidekiq
```

### Emails not sending

```bash
# Test SMTP from rails console
docker compose run --rm rails bundle exec rails console

# In console:
# ActionMailer::Base.mail(from: 'test@test.com', to: 'your@email.com', subject: 'Test', body: 'Hello').deliver_now
```

Check SMTP variables in `.env`: `SMTP_ADDRESS`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`.

### API returns 401 Unauthorized

```bash
# Verify the token is correct
# Go to Chatwoot → Profile → Access Token

# If using Nginx, ensure this line exists in nginx config:
# underscores_in_headers on;
# Without this, the api_access_token header is silently dropped
```

### Super Admin lost password

```bash
docker compose run --rm rails bundle exec rails console

# In console:
# user = User.find_by(email: 'admin@example.com')
# user.update!(password: 'new_secure_password')
# exit
```

---

## 5. Nginx / SSL Issues

### 502 Bad Gateway

```bash
# Check if Rails is running
docker compose ps rails
curl -I http://127.0.0.1:3000/api

# If rails is down, check logs
docker compose logs rails

# If rails is up but 502 persists, check nginx config
sudo nginx -t
```

### SSL certificate not renewing

```bash
# Test renewal
sudo certbot renew --dry-run

# If it fails, check port 80 is open
sudo ufw status
sudo ufw allow 80/tcp

# Force renewal
sudo certbot renew --force-renewal
```

### WebSocket connection failing

Ensure nginx config has WebSocket headers:

```nginx
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 86400;
```

---

## 6. Performance Issues

### High memory usage

```bash
# Check container resource usage
docker stats

# Reduce Sidekiq concurrency in .env
SIDEKIQ_CONCURRENCY=3  # default is 10

# Reduce Rails threads
RAILS_MAX_THREADS=3  # default is 5

# Restart
docker compose down && docker compose up -d
```

### Slow responses

```bash
# Check database size
docker compose exec postgres psql -U postgres -c "SELECT pg_size_pretty(pg_database_size('chatwoot_production'));"

# Check for long-running queries
docker compose exec postgres psql -U postgres -c "SELECT pid, now() - pg_stat_activity.query_start AS duration, query FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC LIMIT 5;"

# Vacuum database
docker compose exec postgres psql -U postgres -c "VACUUM ANALYZE;" chatwoot_production
```

### Disk space running out

```bash
# Check disk usage
df -h

# Check Docker disk usage
docker system df

# Clean unused Docker data
docker system prune -f

# Check backup folder size
du -sh ~/chatwoot/backups/
```
