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
| **Media Proxy** | Nginx (sidecar) | Solução local para entrega de mídia |
| **Orquestração** | Docker Compose | Gerenciamento de containers e rede |

## Funcionalidades

- Roteamento multi-departamento (Comercial, Suporte, Financeiro) em um único número WhatsApp
- Mensagens bidirecionais: texto, imagens, áudio, vídeo e documentos
- Painel de atendentes com atribuição de conversas, etiquetas e respostas prontas
- Importação automática de contatos e merge de números brasileiros
- Branding personalizado (logo, favicons) com a identidade do cliente
- Comunicação entre serviços via rede Docker compartilhada (`villanova-net`)
- Media proxy sidecar para resolver validação de URLs `localhost` em desenvolvimento local

## Estrutura do Projeto

```
villanova-chatwoot/
├── chatwoot/
│   ├── docker-compose.yaml      # Stack Chatwoot (Rails, Sidekiq, Postgres, Redis)
│   ├── .env.example             # Template de variáveis de ambiente
│   └── favicons/                # Favicons customizados para o painel
├── evolution-api/
│   ├── docker-compose.yaml      # Stack Evolution API (API, Media Proxy, Manager, Postgres, Redis)
│   ├── .env.example             # Template de variáveis de ambiente
│   ├── media-proxy.conf         # Nginx: proxy de 127.0.0.1:3000 → chatwoot-rails-1:3000
│   ├── manager-nginx.conf       # Correção do bug do nginx do Evolution Manager oficial
│   └── qrcode.html              # Página auxiliar para escanear QR code do WhatsApp
├── docs/
│   ├── setup-local.md           # Guia completo de setup local
│   └── docker.md                # Referência de comandos Docker
├── image/                       # Favicons customizados
└── CLAUDE.md                    # Contexto do projeto para desenvolvimento assistido por IA
```

## Primeiros Passos

### Pré-requisitos

- Docker Desktop instalado e rodando
- Portas livres: `3000`, `5432`, `6379`, `8080`, `9615`

### 1. Criar a rede compartilhada

```bash
docker network create villanova-net
```

### 2. Subir o Chatwoot

```bash
cd chatwoot
cp .env.example .env   # Edite com suas credenciais
docker compose up -d
```

Aguarde ~60 segundos para o Rails iniciar, depois acesse http://127.0.0.1:3000.

### 3. Subir a Evolution API

```bash
cd evolution-api
cp .env.example .env   # Edite com suas credenciais
docker compose up -d
```

Verifique em http://localhost:8080 (Swagger em http://localhost:8080/docs).

### 4. Criar instância WhatsApp

```bash
curl -X POST http://localhost:8080/instance/create \
  -H "apikey: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "villanova-whatsapp",
    "integration": "WHATSAPP-BAILEYS",
    "qrcode": true,
    "chatwootAccountId": "1",
    "chatwootToken": "SEU_CHATWOOT_ACCESS_TOKEN",
    "chatwootUrl": "http://chatwoot-rails-1:3000",
    "chatwootSignMsg": true,
    "chatwootReopenConversation": true,
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Villa Nova",
    "chatwootImportContacts": true,
    "chatwootMergeBrazilContacts": true
  }'
```

### 5. Conectar WhatsApp

```bash
curl http://localhost:8080/instance/connect/villanova-whatsapp \
  -H "apikey: SUA_API_KEY"
```

Escaneie o QR code retornado com o WhatsApp no celular.

> Para instruções detalhadas de setup, troubleshooting e comandos Docker, veja [docs/setup-local.md](docs/setup-local.md) e [docs/docker.md](docs/docker.md).

## Variáveis de Ambiente

Ambos os serviços precisam de arquivos `.env`. Copie os exemplos e preencha seus valores:

- `chatwoot/.env.example` — Configuração do Chatwoot (Rails, Postgres, Redis, SMTP)
- `evolution-api/.env.example` — Configuração da Evolution API (auth, banco, integração Chatwoot)

> **Importante:** O `FRONTEND_URL` no `.env` do Chatwoot deve usar `http://127.0.0.1:3000` (não `localhost`) para desenvolvimento local. O hostname `localhost` falha na validação de URL do `class-validator` da Evolution API, quebrando o envio de mídia. Em produção com domínios reais, isso não é um problema.

## Arquitetura

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

Desenvolvido com assistência de IA usando [Claude Code](https://claude.com/claude-code)
