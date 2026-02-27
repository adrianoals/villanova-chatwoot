# VillaNova — Projeto Chatwoot Multi-Departamento

## Sobre o Projeto

Sistema de multiatendimento via WhatsApp para a empresa Villa Nova Condomínios, baseado em Chatwoot self-hosted.

- **Cliente:** Villa Nova Condomínios — https://villanovacondominios.com.br
- **Escopo técnico:** `escopo_tecnico_chatwoot_multidepartamento.docx`

## Stack Local (Docker)

| Serviço | Container | Porta |
|---------|-----------|-------|
| Chatwoot (Rails) | chatwoot-rails-1 | 127.0.0.1:3000 |
| Chatwoot (Sidekiq) | chatwoot-sidekiq-1 | — |
| PostgreSQL | chatwoot-postgres-1 | 127.0.0.1:5432 |
| Redis | chatwoot-redis-1 | 127.0.0.1:6379 |

### Comandos essenciais

```bash
cd ~/Desktop/VillaNova/chatwoot

# Parar tudo
docker compose down

# Iniciar tudo
docker compose up -d

# Ver status
docker compose ps

# Ver logs
docker compose logs -f rails

# Backup do banco
docker compose exec postgres pg_dump -U postgres chatwoot_production > backup.sql
```

### Credenciais locais (somente teste)

- **Banco:** postgres / chatwoot_pg_2026 / chatwoot_production
- **Redis:** chatwoot_redis_2026
- **Painel:** http://localhost:3000
- **Super Admin:** http://localhost:3000/super_admin

## Branding

- **Logo:** https://villanovacondominios.com.br/wp-content/uploads/2022/11/cropped-Logo-17.jpg
- **Nome:** Villa Nova
- **Favicons:** `chatwoot/favicons/` (copiados manualmente para o container)

Para alterar branding, usar Rails runner (nunca SQL direto):

```bash
docker compose exec -T rails bundle exec rails runner '
  InstallationConfig.find_by(name: "LOGO").update!(
    serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: "URL_DO_LOGO")
  )
'
docker compose exec redis redis-cli -a chatwoot_redis_2026 FLUSHALL
docker compose restart rails sidekiq
```

## Estrutura do Projeto

```
VillaNova/
├── CLAUDE.md                    # Este arquivo
├── .gitignore
├── chatwoot/
│   ├── docker-compose.yaml      # Stack Docker
│   ├── .env                     # Variáveis de ambiente (NÃO comitar)
│   └── favicons/                # Favicons customizados
├── docs/
│   └── .env                     # Outro env (verificar)
├── .claude/
│   └── skills/
│       ├── chatwoot/            # Skill: API do Chatwoot (curl)
│       └── chatwoot-install/    # Skill: Instalação do Chatwoot
└── escopo_tecnico_chatwoot_multidepartamento.docx
```

## Decisões Técnicas

- **API WhatsApp:** Ainda não decidido (Evolution API vs MegaAPI vs API Oficial Meta)
- **Automações:** Python (FastAPI) em vez de N8N — mais leve e mais controle
- **VPS:** Hostinger disponível para testes, provedor final a definir com o cliente
- **Instalação VPS:** Plano é usar Claude Code direto na VPS para instalação

## Próximos Passos

1. Explorar configurações do Chatwoot (departamentos, inboxes, teams)
2. Decidir API WhatsApp
3. Testar na VPS Hostinger
4. Montar automações Python
5. Apresentar ao cliente

## Cuidados

- **Não comitar `.env`** — contém senhas
- **Favicons se perdem** se recriar o container — na produção usar volume mount
- **Redis do Homebrew** pode conflitar na porta 6379 — parar com `brew services stop redis`
