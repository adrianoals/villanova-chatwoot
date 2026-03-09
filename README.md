# VillaNova Chatwoot — Multi-Department WhatsApp Support

🇧🇷 [Leia em Português](README.pt-br.md)

**Built for [Villa Nova Condomínios](https://villanovacondominios.com.br) by XNAP**

A self-hosted multi-department customer support system powered by WhatsApp. Combines Chatwoot (open-source helpdesk) with Evolution API (WhatsApp integration) to enable departments like Sales, Support, and Finance to manage conversations from a single WhatsApp number through a unified dashboard.

## Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Helpdesk** | [Chatwoot](https://www.chatwoot.com/) (self-hosted) | Agent dashboard, conversation routing, teams |
| **WhatsApp API** | [Evolution API](https://github.com/EvolutionAPI/evolution-api) v2 | WhatsApp connection via Baileys (QR code, no Meta costs) |
| **Databases** | PostgreSQL 15/16 + Redis | Data persistence and caching |
| **Media Proxy** | Nginx (sidecar) | Local dev workaround for media delivery |
| **Orchestration** | Docker Compose | Container management and networking |

## Features

- Multi-department routing (Sales, Support, Finance) from a single WhatsApp number
- Full two-way messaging: text, images, audio, video, and documents
- Agent dashboard with conversation assignment, labels, and canned responses
- Automatic contact import and Brazilian phone number merging
- Custom branding (logo, favicons) for the client's identity
- Inter-service communication via shared Docker network (`villanova-net`)
- Media proxy sidecar to handle `localhost` URL validation issues in local development

## Project Structure

```
villanova-chatwoot/
├── chatwoot/
│   ├── docker-compose.yaml      # Chatwoot stack (Rails, Sidekiq, Postgres, Redis)
│   ├── .env.example             # Environment variables template
│   └── favicons/                # Custom favicons for the dashboard
├── evolution-api/
│   ├── docker-compose.yaml      # Evolution API stack (API, Media Proxy, Manager, Postgres, Redis)
│   ├── .env.example             # Environment variables template
│   ├── media-proxy.conf         # Nginx: proxies 127.0.0.1:3000 → chatwoot-rails-1:3000
│   ├── manager-nginx.conf       # Fix for official Evolution Manager nginx bug
│   └── qrcode.html              # Helper page to scan the WhatsApp QR code
├── docs/
│   ├── setup-local.md           # Full local setup walkthrough
│   └── docker.md                # Docker commands reference
├── image/                       # Custom favicon assets
└── CLAUDE.md                    # Project context for AI-assisted development
```

## Getting Started

### Prerequisites

- Docker Desktop installed and running
- Free ports: `3000`, `5432`, `6379`, `8080`, `9615`

### 1. Create the shared network

```bash
docker network create villanova-net
```

### 2. Start Chatwoot

```bash
cd chatwoot
cp .env.example .env   # Edit with your credentials
docker compose up -d
```

Wait ~60 seconds for Rails to boot, then open http://127.0.0.1:3000.

### 3. Start Evolution API

```bash
cd evolution-api
cp .env.example .env   # Edit with your credentials
docker compose up -d
```

Verify at http://localhost:8080 (Swagger docs at http://localhost:8080/docs).

### 4. Create WhatsApp instance

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "villanova-whatsapp",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "YOUR_CHATWOOT_ACCESS_TOKEN",
    "chatwootUrl": "http://chatwoot-rails-1:3000",
    "chatwootSignMsg": true,
    "chatwootReopenConversation": true,
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Villa Nova",
    "chatwootImportContacts": true,
    "chatwootMergeBrazilContacts": true
  }'
```

### 5. Connect WhatsApp

```bash
curl http://localhost:8080/instance/connect/villanova-whatsapp \
  -H "apikey: YOUR_API_KEY"
```

Scan the returned QR code with WhatsApp on your phone.

> For detailed setup instructions, troubleshooting, and Docker commands, see [docs/setup-local.md](docs/setup-local.md) and [docs/docker.md](docs/docker.md).

## Environment Variables

Both services require `.env` files. Copy the examples and fill in your values:

- `chatwoot/.env.example` — Chatwoot configuration (Rails, Postgres, Redis, SMTP)
- `evolution-api/.env.example` — Evolution API configuration (auth, database, Chatwoot integration)

> **Important:** `FRONTEND_URL` in the Chatwoot `.env` must use `http://127.0.0.1:3000` (not `localhost`) for local development. The `localhost` hostname fails URL validation in Evolution API's `class-validator`, breaking media delivery. In production with real domains, this is not an issue.

## Architecture

```
┌──────────────────┐       ┌──────────────────┐
│     Chatwoot      │       │   Evolution API   │
│  (Rails + Sidekiq)│◄─────►│  (WhatsApp Bridge)│
│   :3000           │       │   :8080           │
└────────┬─────────┘       └────────┬─────────┘
         │                          │
    villanova-net              villanova-net
         │                          │
   ┌─────┴─────┐             ┌─────┴──────┐
   │ PostgreSQL │             │ PostgreSQL  │
   │   + Redis  │             │   + Redis   │
   └───────────┘             └────────────┘
                                    │
                              ┌─────┴─────┐
                              │  WhatsApp  │
                              │  (Baileys) │
                              └───────────┘
```

---

Built with AI-assisted development using [Claude Code](https://claude.com/claude-code)
