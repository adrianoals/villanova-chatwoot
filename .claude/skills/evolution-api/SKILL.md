---
name: evolution-api
description: >
  Evolution API — open-source WhatsApp integration API (v2). Use this skill whenever the user wants to:
  install or configure Evolution API, create/manage WhatsApp instances, integrate Evolution API with Chatwoot,
  send or receive WhatsApp messages via API, configure webhooks, handle media (images, audio, video, documents),
  manage groups, set up Docker Compose for Evolution API, configure Nginx/SSL, troubleshoot connection issues,
  or work with WhatsApp automation. Also trigger when the user mentions "Evolution API", "evolution-api",
  "conectar WhatsApp", "API do WhatsApp", "instância WhatsApp", "QR code WhatsApp", or any WhatsApp API
  integration task — even if they don't explicitly say "Evolution API".
---

# Evolution API v2 — Skill Reference

Evolution API is an open-source REST API that connects to WhatsApp (via Baileys library or official Meta Cloud API) and exposes endpoints for sending/receiving messages, managing instances, and integrating with platforms like Chatwoot, Typebot, n8n, OpenAI, and Dify.

- **Current version:** v2.3.7 (December 2025)
- **Docker image:** `evoapicloud/evolution-api:latest`
- **Frontend (Manager):** `evoapicloud/evolution-manager:latest` (port 3000)
- **Default port:** 8080
- **Swagger docs:** `http://<server>:8080/docs`
- **Runtime:** Node.js 24 (Alpine)
- **GitHub:** https://github.com/EvolutionAPI/evolution-api (~7,300 stars, 155+ contributors)
- **Docs:** https://doc.evolution-api.com
- **License:** Apache 2.0

## Quick Start — Docker Compose

Minimal `docker-compose.yaml` to run Evolution API with PostgreSQL, Redis and Manager:

```yaml
services:
  evolution-api:
    container_name: evolution_api
    image: evoapicloud/evolution-api:latest
    restart: always
    depends_on:
      - evolution-redis
      - evolution-postgres
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - evolution_instances:/evolution/instances
    env_file:
      - .env
    networks:
      - evolution-net

  evolution-manager:
    container_name: evolution_manager
    image: evoapicloud/evolution-manager:latest
    restart: always
    ports:
      - "127.0.0.1:9615:80"
    networks:
      - evolution-net

  evolution-redis:
    container_name: evolution_redis
    image: redis:latest
    restart: always
    command: redis-server --port 6379 --appendonly yes
    volumes:
      - evolution_redis:/data
    networks:
      - evolution-net

  evolution-postgres:
    container_name: evolution_postgres
    image: postgres:15
    restart: always
    command: ["postgres", "-c", "max_connections=1000"]
    environment:
      - POSTGRES_DB=evolution_db
      - POSTGRES_USER=evolution
      - POSTGRES_PASSWORD=evolution_pass_2026
    volumes:
      - evolution_pgdata:/var/lib/postgresql/data
    networks:
      - evolution-net

volumes:
  evolution_instances:
  evolution_redis:
  evolution_pgdata:

networks:
  evolution-net:
    driver: bridge
```

Minimal `.env`:

```env
SERVER_URL=http://localhost:8080
# WARNING: In Docker, if Chatwoot uses FRONTEND_URL=http://localhost:3000,
# media sending will FAIL because class-validator's isURL() rejects 'localhost' (no TLD).
# Use http://127.0.0.1:3000 for FRONTEND_URL in the Chatwoot .env instead.
AUTHENTICATION_API_KEY=YOUR_GLOBAL_API_KEY_HERE

# Database
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://evolution:evolution_pass_2026@evolution-postgres:5432/evolution_db?schema=evolution_api

# Redis
CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://evolution-redis:6379/6

# Chatwoot (enable if integrating)
CHATWOOT_ENABLED=true
```

> For the full list of environment variables, see `references/env-variables.md`

## Authentication

All API requests require the header:

```
apikey: YOUR_GLOBAL_API_KEY
```

This is the value of `AUTHENTICATION_API_KEY` in your `.env`.

Individual instances can also have their own tokens (set during creation).

## Core Workflow

### 1. Create an Instance

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "my-whatsapp",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true
  }'
```

The response includes a QR code (base64) if `qrcode: true`.

### 2. Connect (Get QR Code)

```bash
curl -X GET http://localhost:8080/instance/connect/my-whatsapp \
  -H "apikey: YOUR_API_KEY"
```

Returns a QR code to scan with WhatsApp on your phone.

### 3. Check Connection State

```bash
curl -X GET http://localhost:8080/instance/connectionState/my-whatsapp \
  -H "apikey: YOUR_API_KEY"
```

Response: `{ "state": "open" }` when connected.

### 4. Send a Text Message

```bash
curl -X POST http://localhost:8080/message/sendText/my-whatsapp \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "text": "Hello from Evolution API!"
  }'
```

### 5. Send Media

```bash
curl -X POST http://localhost:8080/message/sendMedia/my-whatsapp \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "5511999999999",
    "mediatype": "image",
    "caption": "Check this out!",
    "media": "https://example.com/image.png"
  }'
```

`mediatype` options: `image`, `video`, `document`

> For ALL message types and endpoints, see `references/api-endpoints.md`

## Chatwoot Integration

Evolution API has **native Chatwoot integration** — it can automatically create an inbox in Chatwoot and route all WhatsApp messages through it.

### Prerequisites

1. Chatwoot running and accessible
2. A Chatwoot **admin user API access token** (from Profile Settings > Access Token)
3. The Chatwoot **account ID** (from the URL: `/app/accounts/ACCOUNT_ID/...`)

### Configure via API

```bash
curl -X POST http://localhost:8080/chatwoot/set/my-whatsapp \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "accountId": "1",
    "token": "CHATWOOT_ADMIN_ACCESS_TOKEN",
    "url": "https://chatwoot.example.com",
    "nameInbox": "WhatsApp",
    "signMsg": true,
    "reopenConversation": true,
    "conversationPending": true,
    "importContacts": true,
    "importMessages": true,
    "daysLimitImportMessages": 30,
    "autoCreate": true,
    "organization": "My Company",
    "mergeBrazilContacts": true
  }'
```

### Or Configure During Instance Creation

Pass the Chatwoot parameters directly in the `/instance/create` body:

```json
{
  "instanceName": "my-whatsapp",
  "integration": "WHATSAPP-BAILEYS",
  "qrcode": true,
  "chatwootAccountId": "1",
  "chatwootToken": "CHATWOOT_ADMIN_ACCESS_TOKEN",
  "chatwootUrl": "https://chatwoot.example.com",
  "chatwootSignMsg": true,
  "chatwootReopenConversation": true,
  "chatwootConversationPending": true,
  "chatwootAutoCreate": true,
  "chatwootNameInbox": "WhatsApp",
  "chatwootImportContacts": true,
  "chatwootImportMessages": true,
  "chatwootDaysLimitImportMessages": 30,
  "chatwootMergeBrazilContacts": true
}
```

### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `autoCreate` | If `true`, Evolution API creates the inbox automatically in Chatwoot |
| `signMsg` | Adds the agent's name as signature in outgoing messages |
| `reopenConversation` | Reopen the same conversation instead of creating a new one |
| `conversationPending` | New conversations start as "Pending" (requires manual assignment) |
| `mergeBrazilContacts` | Merges Brazilian numbers with/without the 9th digit |
| `importContacts` | Import existing WhatsApp contacts into Chatwoot |
| `importMessages` | Import message history into Chatwoot |

> For detailed Chatwoot integration guide, see `references/chatwoot-integration.md`

## Webhooks

Configure per-instance webhooks to receive events:

```bash
curl -X POST http://localhost:8080/webhook/set/my-whatsapp \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "url": "https://your-server.com/webhook",
    "byEvents": false,
    "base64": false,
    "events": [
      "MESSAGES_UPSERT",
      "CONNECTION_UPDATE",
      "QRCODE_UPDATED"
    ]
  }'
```

Key events: `MESSAGES_UPSERT` (new message), `CONNECTION_UPDATE` (connection status), `QRCODE_UPDATED` (QR code changed), `SEND_MESSAGE` (message sent), `MESSAGES_UPDATE` (delivery/read status).

> For the full webhook events reference, see `references/api-endpoints.md`

## Instance Management

| Action | Method | Endpoint |
|--------|--------|----------|
| Create | `POST` | `/instance/create` |
| Connect (QR) | `GET` | `/instance/connect/{name}` |
| Status | `GET` | `/instance/connectionState/{name}` |
| List all | `GET` | `/instance/fetchInstances` |
| Restart | `PUT` | `/instance/restart/{name}` |
| Logout | `DELETE` | `/instance/logout/{name}` |
| Delete | `DELETE` | `/instance/delete/{name}` |

## Nginx Reverse Proxy (Production)

```nginx
server {
  server_name api.empresa.com.br;

  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }
}
```

SSL with Certbot:
```bash
snap install --classic certbot
certbot --nginx -d api.empresa.com.br
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| QR code not appearing | Check `QRCODE_LIMIT` env var, restart the instance |
| Instance disconnects | Check `DEL_INSTANCE` env var (set to `false`), verify Redis is running |
| Chatwoot not receiving messages | Verify `CHATWOOT_ENABLED=true`, check Chatwoot URL is reachable from the container |
| 401 Unauthorized | Check `apikey` header matches `AUTHENTICATION_API_KEY` |
| Messages not sending | Check connection state is `open`, verify the number format (country code + number) |
| Media not sending | Ensure the URL is publicly accessible or use base64. **In local Docker dev:** `class-validator`'s `isURL()` rejects `localhost` URLs (no TLD) — use `127.0.0.1` instead |

> For detailed troubleshooting, see `references/troubleshooting.md`

## Reference Files

- `references/env-variables.md` — Complete list of all environment variables
- `references/api-endpoints.md` — All API endpoints with request/response examples
- `references/chatwoot-integration.md` — Step-by-step Chatwoot integration guide
- `references/troubleshooting.md` — Common issues and solutions
