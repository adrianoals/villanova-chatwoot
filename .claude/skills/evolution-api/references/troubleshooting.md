# Evolution API — Troubleshooting Guide

## Connection Issues

### QR Code not appearing
1. Check `QRCODE_LIMIT` env var (default: 30)
2. Restart the instance:
   ```bash
   curl -X POST http://localhost:8080/instance/restart/INSTANCE_NAME \
     -H "apikey: YOUR_API_KEY"
   ```
3. Delete and recreate the instance if persistent
4. Check logs: `docker compose logs -f evolution-api`

### Instance disconnects frequently
1. Set `DEL_INSTANCE=false` in `.env` to prevent auto-deletion
2. Check Redis is running and healthy
3. Verify internet stability on the server
4. Check if the WhatsApp account is being used on another device/browser
5. Avoid running WhatsApp Web on a browser simultaneously

### Connection state stuck on "connecting"
1. Logout and reconnect:
   ```bash
   curl -X DELETE http://localhost:8080/instance/logout/INSTANCE_NAME \
     -H "apikey: YOUR_API_KEY"
   curl -X GET http://localhost:8080/instance/connect/INSTANCE_NAME \
     -H "apikey: YOUR_API_KEY"
   ```
2. If persistent, delete and recreate the instance

### WhatsApp banned the number
- This can happen with unofficial APIs (Baileys)
- Mitigation: don't send bulk messages, avoid spam patterns
- Consider migrating to WhatsApp Business Cloud API (`WHATSAPP-BUSINESS` integration)

---

## API Issues

### 401 Unauthorized
- Check `apikey` header matches `AUTHENTICATION_API_KEY` in `.env`
- Header format: `apikey: YOUR_KEY` (not `Authorization: Bearer`)

### 404 Not Found
- Check the instance name in the URL is correct
- Verify the instance exists: `GET /instance/fetchInstances`

### Messages not sending
1. Check connection state is `open`:
   ```bash
   curl -X GET http://localhost:8080/instance/connectionState/INSTANCE_NAME \
     -H "apikey: YOUR_API_KEY"
   ```
2. Verify number format: use country code without `+` (e.g., `5511999999999`)
3. Check if the number is a valid WhatsApp number:
   ```bash
   curl -X POST http://localhost:8080/chat/whatsappNumbers/INSTANCE_NAME \
     -H "apikey: YOUR_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{ "numbers": ["5511999999999"] }'
   ```

### Media not sending
- If using URL: ensure the URL is publicly accessible
- If using base64: ensure the string is valid and includes proper mimetype
- Check file size limits (WhatsApp limits: 16MB images, 64MB video, 100MB documents)
- For audio: WhatsApp expects opus codec in ogg container

---

## Chatwoot Integration Issues

### Inbox not created automatically
1. Verify `autoCreate: true` (or `chatwootAutoCreate: true`)
2. Check the Chatwoot URL is reachable from the Evolution API container
3. Verify the token has admin permissions
4. Check Evolution API logs for errors

### Messages not appearing in Chatwoot
1. Verify `CHATWOOT_ENABLED=true` in `.env`
2. Check instance connection state is `open`
3. Verify Chatwoot inbox exists and is active
4. Check network connectivity between containers

### Agent replies not reaching WhatsApp
1. Check the Chatwoot inbox webhook URL points to Evolution API
   - Should be: `http://evolution-api:8080/chatwoot/webhook/INSTANCE_NAME`
2. Verify network connectivity from Chatwoot to Evolution API
3. Check Chatwoot Sidekiq is running (webhooks are processed by background jobs)

### Duplicate contacts (Brazilian numbers)
Enable `mergeBrazilContacts: true` — this handles the 9th digit variations in Brazilian phone numbers.

### QR code not showing in Chatwoot
- Set `CHATWOOT_BOT_CONTACT=true` in `.env`
- The QR code appears as a message from a bot contact in the inbox

### Old messages not imported
1. Check `importMessages: true` and `daysLimitImportMessages` value
2. For faster import, use direct DB connection:
   ```env
   CHATWOOT_IMPORT_DATABASE_CONNECTION_URI=postgresql://user:pass@chatwoot-db:5432/chatwoot_production
   ```

---

## Docker Issues

### Container won't start
1. Check logs: `docker compose logs -f evolution-api`
2. Verify `.env` file exists and has required variables
3. Check PostgreSQL and Redis are healthy:
   ```bash
   docker compose ps
   ```

### Database connection errors
1. Verify `DATABASE_CONNECTION_URI` format:
   ```
   postgresql://user:password@host:5432/database?schema=evolution_api
   ```
2. Check PostgreSQL container is running and healthy
3. Ensure the database exists

### Redis connection errors
1. Verify `CACHE_REDIS_URI` format: `redis://host:6379/6`
2. If Redis has a password: `redis://:password@host:6379/6`
3. Check Redis container is running

### Port conflicts
- Default port 8080 may conflict with other services
- Change in `.env`: `SERVER_PORT=8081`
- Update docker-compose port mapping accordingly

### Data persistence
- Instance data is stored in the `evolution_instances` volume
- Database data in `postgres_data` volume
- Redis data in `evolution_redis` volume
- Losing these volumes means losing all instances and message history

---

## Performance Issues

### Slow response times
1. Enable Redis cache: `CACHE_REDIS_ENABLED=true`
2. Increase database max connections if needed
3. Check server resources (CPU, RAM, disk I/O)

### High memory usage
1. Reduce `EVENT_EMITTER_MAX_LISTENERS` if not needed
2. Set `LOG_LEVEL` to `ERROR,WARN` only
3. Disable unnecessary database saves (e.g., `DATABASE_SAVE_DATA_HISTORIC=false`)

### Webhook delays
1. Check `WEBHOOK_REQUEST_TIMEOUT_MS` (default 60000ms)
2. Ensure webhook receiver responds quickly (< 5 seconds)
3. Check retry configuration if webhooks are failing

---

## Nginx / SSL Issues

### SSL certificate not working
1. Ensure DNS is pointing to the server
2. Check Nginx config has correct `server_name`
3. For Let's Encrypt, ensure port 80 is open for verification

### Websocket errors behind Nginx
Add to Nginx config:
```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

---

## Docker Networking (Chatwoot + Evolution API)

### `host.docker.internal` not resolving (ENOTFOUND)
On Docker Desktop (Windows/WSL2), `host.docker.internal` can be unreliable. **Recommended approach:**

Create a shared external network so containers communicate directly by name:
```bash
docker network create villanova-net
```

Add to both `docker-compose.yaml` files:
```yaml
# In the services that need cross-stack communication:
networks:
  - default        # keep internal network
  - villanova-net  # shared network

# At the bottom:
networks:
  villanova-net:
    external: true
```

Then use container names as hostnames:
- Evolution API → Chatwoot: `http://chatwoot-rails-1:3000`
- Chatwoot → Evolution API: `http://evolution_api:8080`

### `ENOTFOUND host` when updating message source ID
Known issue in Evolution API v2.3.7 when using container names or `host.docker.internal`. Messages are delivered to WhatsApp successfully but Chatwoot shows "Falha ao enviar". The error occurs specifically in the `updateChatwootMessageSourceId` function. May resolve with real domain names in production.

### Evolution Manager won't start (nginx error)
The official `evoapicloud/evolution-manager:latest` image has a bug in its nginx config:
```
nginx: [emerg] invalid value "must-revalidate" in /etc/nginx/conf.d/nginx.conf:11
```

**Fix:** Create a corrected nginx config and mount it as a volume:
1. Copy the original config, remove `must-revalidate` from the `gzip_proxied` line
2. Mount in docker-compose:
```yaml
evolution-manager:
  volumes:
    - ./manager-nginx.conf:/etc/nginx/conf.d/nginx.conf:ro
```

### Chatwoot inbox webhook URL incorrect
When `autoCreate: true`, Evolution API creates the inbox with a webhook URL based on `SERVER_URL`. If `SERVER_URL` is `localhost`, the Chatwoot container can't reach it. Fix:
1. Set `SERVER_URL` to the container name: `http://evolution_api:8080`
2. Update Chatwoot inbox webhook: `PATCH /api/v1/accounts/{id}/inboxes/{inbox_id}`

---

## Useful Debug Commands

```bash
# Check all instances
curl -s http://localhost:8080/instance/fetchInstances \
  -H "apikey: YOUR_API_KEY"

# Check specific instance connection
curl -s http://localhost:8080/instance/connectionState/INSTANCE_NAME \
  -H "apikey: YOUR_API_KEY"

# Check Chatwoot config for instance
curl -s http://localhost:8080/chatwoot/find/INSTANCE_NAME \
  -H "apikey: YOUR_API_KEY"

# Check webhook config
curl -s http://localhost:8080/webhook/find/INSTANCE_NAME \
  -H "apikey: YOUR_API_KEY"

# Test container-to-container connectivity
docker exec evolution_api wget -qO- --timeout=5 http://chatwoot-rails-1:3000/
docker exec chatwoot-sidekiq-1 wget -qO- --timeout=5 http://evolution_api:8080/

# Check Chatwoot inbox webhook URL
curl -s http://localhost:3000/api/v1/accounts/1/inboxes \
  -H "api_access_token: YOUR_CHATWOOT_TOKEN"

# View Evolution API logs
docker compose logs -f evolution-api --tail 100

# Swagger docs (interactive API explorer)
# Open in browser: http://localhost:8080/docs
```
