# Evolution API + Chatwoot — Integration Guide

Step-by-step guide to integrate Evolution API with Chatwoot for WhatsApp multi-agent support.

## Prerequisites

1. **Chatwoot** running and accessible (e.g., `http://localhost:3000`)
2. **Evolution API** running (e.g., `http://localhost:8080`)
3. Both services must be able to reach each other (same network or public URLs)

## Step 1: Enable Chatwoot in Evolution API

In your Evolution API `.env` file:

```env
CHATWOOT_ENABLED=true
CHATWOOT_MESSAGE_READ=true
CHATWOOT_MESSAGE_DELETE=true
CHATWOOT_BOT_CONTACT=true
```

Restart Evolution API after changing env vars.

## Step 2: Get Chatwoot Credentials

### Admin API Token
1. Log in to Chatwoot
2. Go to **Profile Settings** (click avatar → Profile)
3. Copy the **Access Token** at the bottom

### Account ID
From the Chatwoot URL: `http://localhost:3000/app/accounts/1/...` → Account ID is `1`

## Step 3: Create Instance with Chatwoot (Recommended)

Create the Evolution API instance with Chatwoot params in a single call:

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "whatsapp-comercial",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "YOUR_CHATWOOT_ACCESS_TOKEN",
    "chatwootUrl": "http://chatwoot-rails:3000",
    "chatwootSignMsg": true,
    "chatwootReopenConversation": true,
    "chatwootConversationPending": true,
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Comercial",
    "chatwootImportContacts": true,
    "chatwootImportMessages": true,
    "chatwootDaysLimitImportMessages": 30,
    "chatwootMergeBrazilContacts": true,
    "chatwootOrganization": "Villa Nova",
    "chatwootLogo": "https://example.com/logo.png"
  }'
```

When `chatwootAutoCreate: true`, Evolution API automatically:
- Creates an **API Inbox** in Chatwoot
- Configures the webhook URL
- Creates a bot contact for QR code display

## Step 3 (Alternative): Configure Existing Instance

If the instance already exists:

```bash
curl -X POST http://localhost:8080/chatwoot/set/whatsapp-comercial \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "enabled": true,
    "accountId": "1",
    "token": "YOUR_CHATWOOT_ACCESS_TOKEN",
    "url": "http://chatwoot-rails:3000",
    "nameInbox": "WhatsApp Comercial",
    "signMsg": true,
    "reopenConversation": true,
    "conversationPending": true,
    "autoCreate": true,
    "importContacts": true,
    "importMessages": true,
    "daysLimitImportMessages": 30,
    "mergeBrazilContacts": true,
    "organization": "Villa Nova"
  }'
```

## Step 4: Connect WhatsApp (QR Code)

```bash
curl -X GET http://localhost:8080/instance/connect/whatsapp-comercial \
  -H "apikey: YOUR_API_KEY"
```

Scan the QR code with WhatsApp on your phone.

If `CHATWOOT_BOT_CONTACT=true`, the QR code will also appear as a message in the Chatwoot inbox from a bot contact.

## Step 5: Verify Integration

Check connection state:
```bash
curl -X GET http://localhost:8080/instance/connectionState/whatsapp-comercial \
  -H "apikey: YOUR_API_KEY"
```

Check Chatwoot config:
```bash
curl -X GET http://localhost:8080/chatwoot/find/whatsapp-comercial \
  -H "apikey: YOUR_API_KEY"
```

## Parameters Explained

| Parameter | Type | Description |
|-----------|------|-------------|
| `enabled` | bool | Activate/deactivate the integration |
| `accountId` | string | Chatwoot account ID |
| `token` | string | Admin user API access token |
| `url` | string | Chatwoot base URL (must be reachable from Evolution API container) |
| `nameInbox` | string | Custom inbox name in Chatwoot |
| `signMsg` | bool | Add agent name as signature in messages sent from Chatwoot |
| `signDelimiter` | string | Delimiter between signature and message (default: `\n`) |
| `reopenConversation` | bool | `true` = reopen same conversation, `false` = create new |
| `conversationPending` | bool | `true` = new conversations start as Pending, `false` = Open |
| `mergeBrazilContacts` | bool | Merge Brazilian numbers with/without 9th digit (+55 11 9XXXX vs 11 XXXX) |
| `importContacts` | bool | Import WhatsApp contacts into Chatwoot |
| `importMessages` | bool | Import WhatsApp message history |
| `daysLimitImportMessages` | number | Days to look back for message import |
| `autoCreate` | bool | Auto-create inbox in Chatwoot |
| `organization` | string | Bot contact organization name |
| `logo` | string | Bot contact profile picture URL |
| `ignoreJids` | array | JIDs to ignore (don't forward to Chatwoot) |

## Docker Networking

When running both services in Docker, they need to be on the same network or use proper hostnames:

### Option A: Same docker-compose (recommended for local dev)

Put both services in the same `docker-compose.yaml` and use service names:
- Evolution API uses `http://rails:3000` for Chatwoot URL
- Chatwoot webhook points to `http://evolution-api:8080`

### Option B: Separate docker-compose files

Create an external network:
```bash
docker network create shared-net
```

Add to both docker-compose files:
```yaml
networks:
  default:
    external: true
    name: shared-net
```

### Option C: Public URLs (production)

Use the public domain names:
- `https://chat.empresa.com.br` for Chatwoot
- `https://api.empresa.com.br` for Evolution API

## Multi-Department Setup

For multiple WhatsApp numbers (one per department), create one instance per number:

```bash
# Comercial
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "whatsapp-comercial",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "TOKEN",
    "chatwootUrl": "http://rails:3000",
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Comercial",
    "chatwootConversationPending": true
  }'

# Suporte
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "whatsapp-suporte",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "TOKEN",
    "chatwootUrl": "http://rails:3000",
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Suporte",
    "chatwootConversationPending": true
  }'

# Financeiro
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "whatsapp-financeiro",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "TOKEN",
    "chatwootUrl": "http://rails:3000",
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Financeiro",
    "chatwootConversationPending": true
  }'
```

Each creates a separate Inbox in Chatwoot. Then assign Teams to each Inbox.

## Message Import from Chatwoot DB

To import older messages directly from Chatwoot's database (faster than API):

```env
CHATWOOT_IMPORT_DATABASE_CONNECTION_URI=postgresql://postgres:password@chatwoot-postgres:5432/chatwoot_production
CHATWOOT_IMPORT_PLACEHOLDER_MEDIA_MESSAGE=true
```

## Common Issues

| Issue | Solution |
|-------|----------|
| Inbox not created in Chatwoot | Check `autoCreate: true`, verify Chatwoot URL is reachable, check token has admin permissions |
| Messages not appearing in Chatwoot | Verify `CHATWOOT_ENABLED=true` in .env, check instance connection state is `open` |
| Agent replies not reaching WhatsApp | Check the Chatwoot webhook URL points to Evolution API, verify network connectivity |
| Duplicate contacts (Brazil) | Enable `mergeBrazilContacts: true` |
| QR code not showing in Chatwoot | Enable `CHATWOOT_BOT_CONTACT=true` in .env |
