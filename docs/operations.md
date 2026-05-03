# Operações — Villa Nova Chatwoot

Guia rápido de operação na VPS de produção. Pra setup inicial do servidor (do zero), ver [`vps-production.md`](../.claude/skills/chatwoot-install/references/vps-production.md).

## Acesso

```bash
ssh villanova-vps
# Equivale a: ssh -i ~/.ssh/villanova_vps root@69.62.96.47
cd /opt/villanova
```

## Atualizar serviços (fluxo padrão)

```bash
# No Mac:
git add . && git commit -m "msg" && git push origin main

# Na VPS:
ssh villanova-vps
cd /opt/villanova && git pull
cd <traefik|chatwoot|evolution> && docker compose up -d
```

`docker compose up -d` recria só os containers cujas imagens/configs mudaram.

## Comandos por stack

### Traefik (reverse proxy + SSL)

```bash
cd /opt/villanova/traefik
docker compose ps
docker compose logs -f
docker compose restart traefik       # após mudar labels nos outros stacks
ls letsencrypt/                       # acme.json (certs)
```

### Chatwoot

```bash
cd /opt/villanova/chatwoot
docker compose ps
docker compose logs -f rails
docker compose logs -f sidekiq

# Console Rails
docker compose exec rails bundle exec rails console

# Backup
docker compose exec -T postgres pg_dump -U postgres chatwoot_production \
  | gzip > /opt/villanova/backups/chatwoot_$(date +%Y%m%d_%H%M%S).sql.gz

# Atualizar imagem (cuidado: pode requerer migration)
docker compose pull rails
docker compose run --rm rails bundle exec rails db:chatwoot_prepare
docker compose up -d
```

### Evolution API

```bash
cd /opt/villanova/evolution
docker compose ps
docker compose logs -f evolution-api

# Status da instância WhatsApp
EVO_KEY=$(grep AUTHENTICATION_API_KEY .env | cut -d= -f2)
curl -s -H "apikey: $EVO_KEY" \
  https://evo.villanovacondominios.com.br/instance/connectionState/villanova-whatsapp \
  | python3 -m json.tool
```

## Branding (logo, nome) — Rails runner

```bash
ssh villanova-vps
cd /opt/villanova/chatwoot
docker compose exec -T rails bundle exec rails runner '
  configs = {
    "INSTALLATION_NAME" => "Villa Nova",
    "BRAND_NAME"        => "Villa Nova",
    "LOGO"              => "URL_PUBLICA_DO_LOGO",
    "LOGO_THUMBNAIL"    => "URL_PUBLICA_DO_LOGO",
    "LOGO_DARK"         => "URL_PUBLICA_DO_LOGO"
  }
  configs.each do |name, value|
    config = InstallationConfig.find_or_initialize_by(name: name)
    config.update!(serialized_value: ActiveSupport::HashWithIndifferentAccess.new(value: value))
  end
'
docker compose exec redis sh -c 'redis-cli -a "$REDIS_PASSWORD" FLUSHALL'
docker compose restart rails sidekiq
```

Hard-refresh no browser (`Cmd+Shift+R`) pra ver o logo novo.

## Reset senha admin Chatwoot

```bash
cd /opt/villanova/chatwoot
docker compose run --rm rails bundle exec rails console
# Dentro do console:
# user = User.find_by(email: "admin@example.com")
# user.update!(password: "NovaSenhaForte")
# exit
```

## Reconectar WhatsApp (caso desconecte)

```bash
EVO_KEY=$(ssh villanova-vps 'grep AUTHENTICATION_API_KEY /opt/villanova/evolution/.env | cut -d= -f2')

# Pegar QR
curl -s -H "apikey: $EVO_KEY" \
  https://evo.villanovacondominios.com.br/instance/connect/villanova-whatsapp \
  | python3 -c "import sys, json, base64, re; d=json.load(sys.stdin); b64=re.sub(r'^data:image/[^;]+;base64,', '', d.get('base64','')); open('/tmp/qr.png','wb').write(base64.b64decode(b64))"
open /tmp/qr.png  # mostra no Preview, escaneia com WhatsApp do celular
```

## Ver/editar `.env` (secrets reais ficam só na VPS)

```bash
ssh villanova-vps
cat /opt/villanova/chatwoot/.env
nano /opt/villanova/chatwoot/.env  # editar
cd /opt/villanova/chatwoot && docker compose up -d  # aplicar
```

## Backup remoto (off-site, recomendado)

Sem isso, perda da VPS = perda dos dados. Opções:

- **Hostinger snapshot semanal** (no painel da VPS)
- **Cron diário** copiando `chatwoot/backups/*.sql.gz` pra S3 / Backblaze B2 / outro VPS via `rsync` ou `rclone`
- **Backup manual** quando lembrar (não recomendado pra produção real)

## Logs do sistema

```bash
ssh villanova-vps

# Acessos SSH
journalctl -u ssh --since "1 hour ago"

# Fail2ban (tentativas bloqueadas)
fail2ban-client status sshd

# Firewall
ufw status verbose

# Disco
df -h
docker system df  # quanto cada coisa do Docker ocupa
```

## Quando algo dá errado

1. **HTTPS quebrado:** `cd /opt/villanova/traefik && docker compose logs --tail 50` — ver erro do Let's Encrypt
2. **Mensagens não chegam:** `docker logs evolution_api --tail 50` — ver se WhatsApp está `state: open`
3. **Chatwoot 502:** `docker compose -f /opt/villanova/chatwoot/docker-compose.yaml logs rails` — Rails crashou? Postgres caiu?
4. **Disco cheio:** `docker system prune -a --volumes` (cuidado: limpa imagens não usadas, mas preserva volumes nomeados)
