# Agent Onboarding — Villa Nova Chatwoot

> **Pra qualquer IA / agente novo entrando neste projeto.** Lê isso primeiro antes de qualquer ação.
> Se algo aqui contradiz `CLAUDE.md`, prevalece o `CLAUDE.md` (mais atual).

## TL;DR (30 segundos)

Sistema de **multiatendimento WhatsApp** (Villa Nova Condomínios). Stack:

- **Chatwoot** — helpdesk open-source (painel dos atendentes)
- **Evolution API v2.3.7** — bridge WhatsApp via Baileys (QR code, sem custo Meta)
- **Traefik v3.6** — reverse proxy + SSL Let's Encrypt automático
- **Hostinger VPS** Ubuntu 24.04

**Roda APENAS em produção.** Sem stack de desenvolvimento local. Edita arquivos no Mac → `git push` → SSH na VPS → `git pull` → `docker compose up -d`.

## URLs (produção)

| Serviço | URL |
|---|---|
| Chatwoot | https://multi-chat.villanovacondominios.com.br |
| Evolution API | https://evo.villanovacondominios.com.br |
| Evolution Manager | https://evo.villanovacondominios.com.br/manager |

## Acesso à VPS

```bash
ssh villanova-vps    # alias em ~/.ssh/config
# Por baixo: ssh -i ~/.ssh/villanova_vps root@69.62.96.47
```

- **Auth**: chave SSH apenas (senha desabilitada). Chave dedicada: `~/.ssh/villanova_vps`
- **Código**: `/opt/villanova/` é um clone deste repo (mesmo branch `main`)
- **Firewall**: UFW abre só 22, 80, 443. fail2ban monitora SSH.

## Estrutura (Mac e VPS são espelhos via git)

```
villanova-chatwoot/
├── traefik/             # Reverse proxy v3.6 + Let's Encrypt
│   ├── docker-compose.yaml
│   └── letsencrypt/     # acme.json (gitignored, só na VPS)
├── chatwoot/            # Chatwoot v4.13.0 (Rails + Sidekiq + pgvector + Redis)
│   ├── docker-compose.yaml
│   ├── .env             # gitignored, só na VPS
│   └── favicons/        # 8 PNGs montados como volumes
├── evolution/           # Evolution API v2.3.7 (Postgres + Redis)
│   ├── docker-compose.yaml
│   └── .env             # gitignored, só na VPS
├── docs/
│   ├── agent-onboarding.md   # este arquivo
│   ├── operations.md         # ops do dia a dia
│   └── docker.md             # ponteiros
├── .claude/skills/chatwoot-install/references/vps-production.md  # guia setup zero
├── CLAUDE.md            # contexto detalhado, decisões, bugs conhecidos
└── README.md / README.pt-br.md
```

## Workflow padrão de qualquer mudança

```bash
# Mac:
git add . && git commit -m "msg" && git push

# VPS:
ssh villanova-vps
cd /opt/villanova && git pull
cd <traefik|chatwoot|evolution> && docker compose up -d
```

`docker compose up -d` recria só containers que mudaram.

## Convenções importantes

### 1. Secrets ficam SÓ na VPS

Os `.env` reais estão em `/opt/villanova/<stack>/.env` (gitignored). O repo só tem `.env.example`. Se precisar dos valores:

```bash
ssh villanova-vps 'cat /opt/villanova/chatwoot/.env'
ssh villanova-vps 'cat /opt/villanova/evolution/.env'
```

### 2. Nunca cole senhas/tokens no chat

- Senhas vão em **arquivos locais no Mac** (ex: `~/Documents/Projetos/villanova-chatwoot-vps-backup/`) com `chmod 600`
- Quando criar usuário Chatwoot novo, gera senha aleatória, salva em arquivo, mostra **path** ao usuário (não conteúdo)

### 3. Nada de `docker compose down` em horário comercial

Na produção, evite recreate desnecessário. Use `restart` quando possível, e durante horário comercial peça confirmação ao usuário antes de tocar em containers ativos.

## Gotchas críticos (NÃO esqueça)

### Bug Evolution v2.3.7 — workaround obrigatório

A Evolution v2.3.7 tem um default hardcoded errado em `CHATWOOT_IMPORT_DATABASE_CONNECTION_URI` (hostname literal "host", typo "passwprd"). Sem workaround, toda mensagem do Chatwoot → WhatsApp mostra "Falha ao enviar" (apesar de ser entregue).

**Workaround obrigatório no `evolution/.env`:**
```env
CHATWOOT_IMPORT_DATABASE_CONNECTION_URI=postgres://user:password@hostname:port/dbname
```

Esse é o **placeholder exato** que `isImportHistoryAvailable()` verifica como "não configurado". Não modificar.

**Quando remover:** quando Evolution lançar release com fix oficial. Acompanhar https://github.com/EvolutionAPI/evolution-api/releases.

### FORCE_SSL e URL interna

Chatwoot tem `FORCE_SSL=true`. Chamadas HTTP internas dele são redirecionadas pra HTTPS, mas hostnames internos do Docker (ex: `chatwoot-rails-1`) não têm cert SSL → falha.

**Solução já aplicada:** a Evolution chama Chatwoot pela **URL pública HTTPS** (`https://multi-chat.villanovacondominios.com.br`), não pelo nome interno. Se for criar nova instância Evolution ou mudar URL, manter o domínio público.

### Sem SMTP por enquanto

Convites de agente por email **não funcionam**. Quando criar agentes:
- Use Rails console (`docker compose exec rails bundle exec rails runner ...`)
- Defina senha diretamente, com `confirmed_at: Time.now`
- Senha precisa ter pelo menos 1 caractere especial (`@#$%^&*` etc.)
- Salve credenciais temporárias em arquivo local no Mac, não no chat

## Comandos de saúde rápidos

Pra verificar que tudo tá rodando antes de qualquer mudança:

```bash
# Sync
cd ~/Documents/Projetos/villanova-chatwoot && git status
ssh villanova-vps 'cd /opt/villanova && git status && git log --oneline -3'

# Containers
ssh villanova-vps 'for s in traefik chatwoot evolution; do echo "=== $s ==="; cd /opt/villanova/$s && docker compose ps; done'

# WhatsApp connection
ssh villanova-vps 'EVO_KEY=$(grep AUTHENTICATION_API_KEY /opt/villanova/evolution/.env | cut -d= -f2); curl -s -H "apikey: $EVO_KEY" https://evo.villanovacondominios.com.br/instance/connectionState/villanova-whatsapp | python3 -m json.tool'
```

## Onde está cada coisa de documentação

| Documento | Pra quê |
|---|---|
| `CLAUDE.md` | Contexto detalhado, decisões técnicas, problemas conhecidos atualizados |
| `docs/operations.md` | Ops dia-a-dia (logs, backup, branding, troubleshooting) |
| `.claude/skills/chatwoot-install/references/vps-production.md` | Setup completo de servidor do zero |
| `chatwoot/.env.example` | Template de variáveis Chatwoot |
| `evolution/.env.example` | Template Evolution + comentário sobre o workaround |
| `README.md` / `README.pt-br.md` | Visão geral do projeto |

## Como retomar essa conversa em outro dia

Conversas do Claude Code persistem em `~/.claude/projects/<encoded-path>/<session-id>.jsonl` (não nos servidores Anthropic). Pra retomar:

```bash
cd ~/Documents/Projetos/villanova-chatwoot
claude --resume     # picker interativo
# ou claude --resume <session-id> se souber o ID
```

## Pendências naturais (sem urgência)

1. SMTP (Gmail/SendGrid/SES) — desbloqueia convite de agente por email
2. Backup automatizado off-site (`pg_dump` + `storage_data` → S3/B2)
3. Evolution lançar release com fix do bug — aí remover o workaround
4. Mais departamentos / inboxes (Comercial, Suporte) quando precisar
5. Snapshot semanal pelo painel Hostinger
