# VillaNova — Projeto Chatwoot Multi-Departamento

## Sobre o Projeto

Sistema de multiatendimento via WhatsApp para a empresa Villa Nova Condomínios, baseado em Chatwoot self-hosted + Evolution API. **Roda apenas em produção** — não há setup de desenvolvimento local.

- **Cliente:** Villa Nova Condomínios — https://villanovacondominios.com.br
- **Escopo técnico:** `docs/escopo_tecnico_chatwoot_multidepartamento.docx`

## Topologia de produção

VPS Hostinger acessível via `ssh villanova-vps` (chave dedicada `~/.ssh/villanova_vps`). Código no servidor em `/opt/villanova/` espelha esse repo. Cada `docker-compose.yaml` tem um `.env` irmão (no `.gitignore`) com secrets reais.

| Stack | Container(s) | URL pública |
|-------|--------------|-------------|
| **Traefik v3.6** | `traefik` (80, 443) | — (terminação TLS) |
| **Chatwoot v4.13.0** | `chatwoot-rails-1`, `chatwoot-sidekiq-1`, `chatwoot-postgres-1`, `chatwoot-redis-1` | https://multi-chat.villanovacondominios.com.br |
| **Evolution API v2.3.7** | `evolution_api`, `evolution_postgres`, `evolution_redis` | https://evo.villanovacondominios.com.br |
| **Evolution Manager** | (embedded em `evolution_api`) | https://evo.villanovacondominios.com.br/manager |

Todos compartilham a rede externa Docker `villanova-net`. Traefik descobre serviços via labels.

## Fluxo de trabalho (edição → produção)

```bash
# No Mac:
git add . && git commit -m "..." && git push

# Na VPS:
ssh villanova-vps
cd /opt/villanova && git pull
# Reiniciar o stack afetado:
cd <traefik|chatwoot|evolution> && docker compose up -d
```

`.env` ficam **só na VPS** (gitignored). Pra mudanças neles, edita direto via SSH.

## Comandos essenciais (na VPS)

```bash
ssh villanova-vps

# Status e logs
cd /opt/villanova/<stack> && docker compose ps
cd /opt/villanova/chatwoot && docker compose logs -f rails
cd /opt/villanova/evolution && docker compose logs -f evolution-api

# Backup banco Chatwoot
cd /opt/villanova/chatwoot && docker compose exec -T postgres \
  pg_dump -U postgres chatwoot_production | gzip > /opt/villanova/backups/chatwoot_$(date +%Y%m%d).sql.gz

# Reiniciar tudo
for s in traefik chatwoot evolution; do (cd /opt/villanova/$s && docker compose restart); done
```

## Branding (Chatwoot)

Logo, nome e favicons já aplicados. Pra mudar via Rails runner:

```bash
ssh villanova-vps
cd /opt/villanova/chatwoot && docker compose exec -T rails bundle exec rails runner '
  InstallationConfig.find_by(name: "LOGO").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "URL_DO_LOGO")
  )
  # idem pra INSTALLATION_NAME, BRAND_NAME, LOGO_THUMBNAIL, LOGO_DARK
'
docker compose exec redis sh -c 'redis-cli -a "$REDIS_PASSWORD" FLUSHALL'
docker compose restart rails sidekiq
```

Favicons: arquivos em `chatwoot/favicons/` montados como volumes (sobrevivem a recreations).

## Decisões Técnicas

- **API WhatsApp:** Evolution API v2.3.7 (open-source, self-hosted, gratuita)
- **Integração WhatsApp:** Baileys (via QR code, sem custo da Meta)
- **Reverse proxy:** **Traefik v3.6** (não Nginx+Certbot) — labels descobrem serviços, Let's Encrypt automático
- **VPS:** Hostinger Ubuntu 24.04 (`69.62.96.47`)
- **SSH:** chave-only (senha desabilitada), porta 22 + 80 + 443 via UFW, fail2ban ativo
- **Automações futuras:** FastAPI em Python (planejado, ainda não implementado)

## Problemas Conhecidos / Workarounds

### Bug "Falha ao enviar" da Evolution v2.3.7 — workaround aplicado

**Sintoma:** Mensagens enviadas pelo Chatwoot são entregues no WhatsApp mas o status visual no painel Chatwoot mostra "Falha ao enviar".

**Causa raiz:** `isImportHistoryAvailable()` retorna `true` mesmo sem `CHATWOOT_IMPORT_DATABASE_CONNECTION_URI` configurado, porque a URI cai num default hardcoded bugado (`postgresql://user:passwprd@host:5432/chatwoot...` — typo "passwprd" e hostname literal "host"). Isso faz toda mensagem outbound disparar `updateChatwootMessageSourceId` que tenta resolver "host" via DNS → `getaddrinfo EAI_AGAIN host` → Chatwoot nunca recebe a confirmação → marca como falha.

**Workaround (aplicado):** definir `CHATWOOT_IMPORT_DATABASE_CONNECTION_URI=postgres://user:password@hostname:port/dbname` no `.env` da Evolution. Esse é o placeholder *exato* que a função verifica como "não configurado" — força `isImportHistoryAvailable() === false`, desativando o método bugado.

**Quando remover:** quando Evolution lançar release com fix oficial. Acompanhar https://github.com/EvolutionAPI/evolution-api/releases.

## Próximos Passos

1. **Configurar SMTP** no `chatwoot/.env` (necessário pra convidar agentes, password reset)
2. **Backup automatizado** (cron diário com `pg_dump` + storage_data, off-site recomendado)
3. **Departamentos/teams** no Chatwoot (Comercial, Suporte, Financeiro) via UI
4. **Automações Python (FastAPI)** se necessárias
5. **Apresentar ao cliente**

## Cuidados

- **Nunca comitar `.env`** — secrets reais ficam só na VPS, repo só tem `.env.example`
- **Não rodar `git pull` na VPS sem revisar `git diff`** — verifica o que vai mudar
- **Antes de `docker compose up -d` em produção** — confirma que não vai derrubar conexão WhatsApp ativa no horário comercial
- **SECRET_KEY_BASE** do Chatwoot é o secret mais crítico — perda = sessões/tokens encriptados quebram. Salvar em gerenciador de senhas (1Password / Bitwarden).
