# VillaNova Chatwoot — Suporte Multi-Departamento via WhatsApp

🇺🇸 [Read in English](README.md)

**Desenvolvido para [Villa Nova Condomínios](https://villanovacondominios.com.br) por XNAP**

Sistema de multiatendimento via WhatsApp self-hosted. Combina Chatwoot (helpdesk open-source) com Evolution API (integração WhatsApp) para que departamentos como Comercial, Suporte e Financeiro gerenciem conversas de um único número de WhatsApp em um painel unificado.

## Stack Tecnológica

| Componente | Tecnologia | Finalidade |
|-----------|-----------|---------|
| **Helpdesk** | [Chatwoot](https://www.chatwoot.com/) (self-hosted) | Painel de atendentes, roteamento de conversas, times |
| **API WhatsApp** | [Evolution API](https://github.com/EvolutionAPI/evolution-api) v2 | Conexão WhatsApp via Baileys (QR code, sem custo da Meta) |
| **Bancos de dados** | PostgreSQL 15/16 + Redis | Persistência de dados e cache |
| **Reverse proxy** | [Traefik](https://traefik.io/) v3.6 | Roteamento de tráfego, terminação TLS, certificados Let's Encrypt automáticos |
| **Orquestração** | Docker Compose | Gerenciamento de containers e rede |

## Funcionalidades

- Roteamento multi-departamento (Comercial, Suporte, Financeiro) em um único número WhatsApp
- Mensagens bidirecionais: texto, imagens, áudio, vídeo e documentos
- Painel de atendentes com atribuição de conversas, etiquetas e respostas prontas
- Importação automática de contatos e merge de números brasileiros
- Branding personalizado (logo, favicons) com a identidade do cliente
- Comunicação entre serviços via rede Docker compartilhada (`villanova-net`)
- SSL automático via Let's Encrypt (HTTP challenge do Traefik)

## Estrutura do Projeto

```
villanova-chatwoot/
├── traefik/
│   ├── docker-compose.yaml      # Reverse proxy Traefik v3.6 + Let's Encrypt
│   └── .env.example             # (sem vars obrigatórias por enquanto)
├── chatwoot/
│   ├── docker-compose.yaml      # Stack Chatwoot (Rails, Sidekiq, Postgres, Redis) com labels Traefik
│   ├── .env.example             # Template de variáveis de ambiente
│   └── favicons/                # Favicons customizados montados no container
├── evolution/
│   ├── docker-compose.yaml      # Stack Evolution API (API, Postgres, Redis) com labels Traefik
│   └── .env.example             # Template de variáveis de ambiente
├── docs/
│   └── docker.md                # Referência de comandos Docker
├── .claude/                     # Desenvolvimento assistido por IA (skills, settings)
└── CLAUDE.md                    # Contexto do projeto para IA
```

> **Sem stack de desenvolvimento local.** Esse projeto roda apenas em produção. Edições passam por git → SSH na VPS → `git pull` → restart. Ver [guia de deploy VPS](.claude/skills/chatwoot-install/references/vps-production.md) pro setup completo do servidor.

## Stack de produção na VPS

VPS: `villanova-vps` (`ssh villanova-vps`), código em `/opt/villanova/`.

A estrutura no servidor espelha esse repo. Cada `docker-compose.yaml` daqui tem um `.env` irmão (no gitignore) na VPS com os secrets reais.

| Serviço | URL | Pra que serve |
|---------|-----|---------|
| Chatwoot | https://multi-chat.villanovacondominios.com.br | Painel dos atendentes |
| Evolution API | https://evo.villanovacondominios.com.br | Bridge WhatsApp |
| Evolution Manager | https://evo.villanovacondominios.com.br/manager | UI web pra gerenciar instâncias |

## Fluxo de atualização

```bash
# No seu Mac:
# 1. Edita um docker-compose.yaml ou outro config
git add . && git commit -m "..." && git push

# Na VPS (ssh villanova-vps):
cd /opt/villanova
git pull origin main

# Restart do stack afetado:
cd traefik && docker compose up -d
cd ../chatwoot && docker compose up -d
cd ../evolution && docker compose up -d
```

## Variáveis de Ambiente

Cada stack na VPS tem seu próprio `.env` (gitignored, com secrets reais). Templates versionados aqui:

- `chatwoot/.env.example` — Configuração do Chatwoot (Rails, Postgres, Redis, SMTP, FRONTEND_URL)
- `evolution/.env.example` — Configuração da Evolution API (auth, banco, integração Chatwoot, workaround do bug)
- `traefik/.env.example` — Traefik (sem vars obrigatórias por enquanto)

Pra operação dia a dia, veja [docs/operations.md](docs/operations.md).

## Arquitetura

### Produção (VPS)

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
│  :3000 (interno)  │    │  :8080 (interno)  │
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

Traefik termina TLS, roteia por subdomínio e emite certificados Let's Encrypt automaticamente. Portas do Chatwoot e Evolution API **não** ficam expostas publicamente — só Traefik escuta nas 80/443.

---

Desenvolvido com assistência de IA usando [Claude Code](https://claude.com/claude-code)
