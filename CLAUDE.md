# VillaNova — Projeto Chatwoot Multi-Departamento

## Sobre o Projeto

Sistema de multiatendimento via WhatsApp para a empresa Villa Nova Condomínios, baseado em Chatwoot self-hosted + Evolution API.

- **Cliente:** Villa Nova Condomínios — https://villanovacondominios.com.br
- **Escopo técnico:** `escopo_tecnico_chatwoot_multidepartamento.docx`

## Stack Local (Docker)

Todos os serviços compartilham a rede externa `villanova-net` para comunicação direta entre containers.

### Chatwoot (`chatwoot/docker-compose.yaml`)

| Serviço | Container | Porta | Rede |
|---------|-----------|-------|------|
| Chatwoot (Rails) | chatwoot-rails-1 | 127.0.0.1:3000 | default + villanova-net |
| Chatwoot (Sidekiq) | chatwoot-sidekiq-1 | — | default + villanova-net |
| PostgreSQL | chatwoot-postgres-1 | 127.0.0.1:5432 | default |
| Redis | chatwoot-redis-1 | 127.0.0.1:6379 | default |

### Evolution API (`evolution-api/docker-compose.yaml`)

| Serviço | Container | Porta | Rede |
|---------|-----------|-------|------|
| Evolution API | evolution_api | 127.0.0.1:8080 | evolution-net + villanova-net |
| Media Proxy (nginx) | evolution_media_proxy | — (compartilha rede do evolution_api) | network_mode: service:evolution-api |
| Evolution Manager | evolution_manager | 127.0.0.1:9615 | evolution-net |
| PostgreSQL | evolution_postgres | — (interno) | evolution-net |
| Redis | evolution_redis | — (interno) | evolution-net |

### Rede compartilhada

```bash
# Criar a rede (só na primeira vez)
docker network create villanova-net
```

Os serviços se comunicam pelos nomes dos containers:
- Evolution API → Chatwoot: `http://chatwoot-rails-1:3000`
- Chatwoot → Evolution API: `http://evolution_api:8080`

### Comandos essenciais

```bash
# Subir tudo (Chatwoot primeiro, depois Evolution)
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot && docker compose up -d
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api && docker compose up -d

# Parar tudo
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api && docker compose down
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot && docker compose down

# Ver status
docker compose -f chatwoot/docker-compose.yaml ps
docker compose -f evolution-api/docker-compose.yaml ps

# Ver logs
docker compose -f chatwoot/docker-compose.yaml logs -f rails
docker compose -f evolution-api/docker-compose.yaml logs -f evolution-api

# Backup do banco Chatwoot
docker compose -f chatwoot/docker-compose.yaml exec postgres pg_dump -U postgres chatwoot_production > backup.sql
```

### Credenciais locais (somente teste)

**Chatwoot:**
- Banco: postgres / chatwoot_pg_2026 / chatwoot_production
- Redis: chatwoot_redis_2026
- Painel: http://localhost:3000
- Super Admin: http://localhost:3000/super_admin
- API Access Token: `CJH4564njPfz6jhzEuj12fHr`

**Evolution API:**
- API: http://localhost:8080 (Swagger: http://localhost:8080/docs)
- Manager: http://localhost:9615
- API Key: `VillaNova_Evo_2026_TestKey`
- Banco: evolution / evolution_pass_2026 / evolution_db

**Instância WhatsApp:**
- Nome: `villanova-whatsapp`
- Número conectado: 5511994542767
- Integration: WHATSAPP-BAILEYS
- Inbox no Chatwoot: "WhatsApp Villa Nova" (ID: 1)

## Branding

- **Logo:** https://villanovacondominios.com.br/wp-content/uploads/2022/11/cropped-Logo-17.jpg
- **Nome:** Villa Nova
- **Favicons:** `chatwoot/favicons/` (copiados manualmente para o container)

Para alterar branding, usar Rails runner (nunca SQL direto):

```bash
docker compose -f chatwoot/docker-compose.yaml exec -T rails bundle exec rails runner '
  InstallationConfig.find_by(name: "LOGO").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "URL_DO_LOGO")
  )
'
docker compose -f chatwoot/docker-compose.yaml exec redis redis-cli -a chatwoot_redis_2026 FLUSHALL
docker compose -f chatwoot/docker-compose.yaml restart rails sidekiq
```

## Estrutura do Projeto

```
VillaNova-Chatwoot/
├── CLAUDE.md                    # Este arquivo
├── .gitignore
├── chatwoot/
│   ├── docker-compose.yaml      # Stack Chatwoot (Rails, Sidekiq, Postgres, Redis)
│   ├── .env                     # Variáveis de ambiente (NÃO comitar)
│   └── favicons/                # Favicons customizados
├── evolution-api/
│   ├── docker-compose.yaml      # Stack Evolution API (API, Media Proxy, Manager, Postgres, Redis)
│   ├── .env                     # Variáveis de ambiente (NÃO comitar)
│   ├── media-proxy.conf         # Nginx proxy: redireciona localhost:3000 → chatwoot-rails-1:3000 (só local)
│   ├── manager-nginx.conf       # Fix nginx config do Manager (bug imagem oficial)
│   └── qrcode.html              # Página auxiliar para escanear QR code
├── docs/
│   └── docker.md                # Comandos Docker úteis
├── .claude/
│   └── skills/
│       ├── chatwoot/            # Skill: API do Chatwoot (curl)
│       ├── chatwoot-install/    # Skill: Instalação do Chatwoot
│       └── evolution-api/       # Skill: Evolution API v2 (endpoints, env vars, troubleshooting)
└── escopo_tecnico_chatwoot_multidepartamento.docx
```

## Decisões Técnicas

- **API WhatsApp:** Evolution API v2.3.7 (open-source, self-hosted, gratuita)
- **Integração:** WHATSAPP-BAILEYS (via QR code, sem custo da Meta)
- **Automações:** Python (FastAPI) em vez de N8N — mais leve e mais controle
- **VPS:** Hostinger disponível para testes, provedor final a definir com o cliente
- **Instalação VPS:** Plano é usar Claude Code direto na VPS para instalação

## Problemas Conhecidos (Local)

- **"Falha ao enviar" no Chatwoot:** Mensagens enviadas pelo celular chegam no destinatário mas o status no Chatwoot mostra "Falha ao enviar". Erro: `ENOTFOUND host` na Evolution API ao atualizar source ID via `updateChatwootMessageSourceId`. Bug cosmético — as mensagens **são entregues** normalmente. Deve resolver em produção com domínios reais.
- **Media (imagens/áudio/documentos) não enviava do Chatwoot → WhatsApp:** Causa raiz: `class-validator`'s `isURL()` rejeita URLs com `localhost` (sem TLD válido). A Evolution API trata a URL como base64, gerando dados inválidos → erro `Input buffer contains unsupported image format` no Sharp. **Solução:** `FRONTEND_URL=http://127.0.0.1:3000` no `.env` do Chatwoot (IPs passam no `isURL`) + nginx media-proxy sidecar no docker-compose da Evolution API. Em produção com domínios reais, o media-proxy **não é necessário**.
- **Evolution Manager nginx bug:** A imagem oficial tem erro no `gzip_proxied` (valor `must-revalidate` inválido). Resolvido montando `manager-nginx.conf` como volume.
- **Favicons se perdem** se recriar o container — na produção usar volume mount.

## Próximos Passos

1. Configurar departamentos/teams no Chatwoot (Comercial, Suporte, Financeiro)
2. Deploy na VPS Hostinger (Chatwoot + Evolution API + Nginx + SSL)
3. Validar se erro de status resolve em produção
4. Montar automações Python (FastAPI)
5. Apresentar ao cliente

## Cuidados

- **Não comitar `.env`** — contém senhas e tokens
- **Subir Chatwoot antes da Evolution API** — a rede `villanova-net` e o Rails precisam estar up
- **Criar rede `villanova-net`** antes do primeiro `docker compose up`
- **FRONTEND_URL no Chatwoot .env deve ser `http://127.0.0.1:3000`** (local) — `localhost` causa falha no envio de mídia pela Evolution API (bug do `isURL` do class-validator)
- **Após reiniciar evolution-api, recriar media-proxy:** `docker compose up -d --force-recreate media-proxy` (o `network_mode: service:` pode perder binding)
