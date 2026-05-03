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
| **Reverse proxy** | [Traefik](https://traefik.io/) v3.6 | Routes traffic, terminates TLS, auto-issues Let's Encrypt certs |
| **Orchestration** | Docker Compose | Container management and networking |

## Features

- Multi-department routing (Sales, Support, Finance) from a single WhatsApp number
- Full two-way messaging: text, images, audio, video, and documents
- Agent dashboard with conversation assignment, labels, and canned responses
- Automatic contact import and Brazilian phone number merging
- Custom branding (logo, favicons) for the client's identity
- Inter-service communication via shared Docker network (`villanova-net`)
- Auto SSL via Let's Encrypt (Traefik HTTP challenge)

## Project Structure

```
villanova-chatwoot/
├── traefik/
│   ├── docker-compose.yaml      # Traefik v3.6 reverse proxy + Let's Encrypt
│   └── .env.example             # (no required vars yet)
├── chatwoot/
│   ├── docker-compose.yaml      # Chatwoot stack (Rails, Sidekiq, Postgres, Redis) with Traefik labels
│   ├── .env.example             # Environment variables template
│   └── favicons/                # Custom favicons mounted into the container
├── evolution/
│   ├── docker-compose.yaml      # Evolution API stack (API, Postgres, Redis) with Traefik labels
│   └── .env.example             # Environment variables template
├── docs/
│   └── docker.md                # Docker commands reference
├── .claude/                     # AI-assisted development (skills, settings)
└── CLAUDE.md                    # Project context for AI-assisted development
```

> **No local development stack.** This project runs in production only. All edits go through git → SSH to VPS → `git pull` → restart. See [VPS deployment guide](.claude/skills/chatwoot-install/references/vps-production.md) for the full server setup.

## Production stack on the VPS

VPS: `villanova-vps` (`ssh villanova-vps`), code at `/opt/villanova/`.

The structure on the server mirrors this repo layout. Each `docker-compose.yaml` here has a sibling `.env` (gitignored) on the VPS containing real secrets.

| Service | URL | Purpose |
|---------|-----|---------|
| Chatwoot | https://multi-chat.villanovacondominios.com.br | Agent dashboard |
| Evolution API | https://evo.villanovacondominios.com.br | WhatsApp bridge |
| Evolution Manager | https://evo.villanovacondominios.com.br/manager | Web UI for instance management |

## Update workflow

```bash
# On your Mac:
# 1. Edit a docker-compose.yaml or other config
git add . && git commit -m "..." && git push

# On the VPS (ssh villanova-vps):
cd /opt/villanova
git pull origin main

# Restart the affected stack:
cd traefik && docker compose up -d
cd ../chatwoot && docker compose up -d
cd ../evolution && docker compose up -d
```

> For day-to-day operations on the VPS, see [docs/operations.md](docs/operations.md). For first-time setup of a new server, see [VPS deployment guide](.claude/skills/chatwoot-install/references/vps-production.md).

## Environment Variables

Each stack on the VPS has its own `.env` file (gitignored, real secrets). Templates are checked in:

- `chatwoot/.env.example` — Chatwoot configuration (Rails, Postgres, Redis, SMTP, FRONTEND_URL)
- `evolution/.env.example` — Evolution API configuration (auth, database, Chatwoot integration, bug workaround)
- `traefik/.env.example` — Traefik (no required vars currently)

## Architecture

### Production (VPS)

```
              Internet (HTTPS)
                    │
            ┌───────┴───────┐
            │    Traefik    │  ← Let's Encrypt (auto)
            │  :80 :443     │
            └───┬───────┬───┘
                │       │
   ┌────────────┘       └────────────┐
   │                                 │
┌──┴────────────────┐    ┌──────────┴────────┐
│     Chatwoot      │    │   Evolution API   │
│  (Rails + Sidekiq)│◄──►│  (WhatsApp Bridge)│
│   :3000 (internal)│    │   :8080 (internal)│
└────────┬──────────┘    └────────┬──────────┘
         │                        │
    villanova-net             villanova-net
         │                        │
   ┌─────┴──────┐           ┌─────┴──────┐
   │ PostgreSQL │           │ PostgreSQL │
   │   + Redis  │           │   + Redis  │
   └────────────┘           └────────────┘
                                  │
                            ┌─────┴─────┐
                            │  WhatsApp │
                            │  (Baileys)│
                            └───────────┘
```

Traefik terminates TLS, routes by subdomain, and auto-issues Let's Encrypt certificates. Chatwoot and Evolution API ports are not exposed publicly — only Traefik listens on 80/443.

---

Built with AI-assisted development using [Claude Code](https://claude.com/claude-code)
