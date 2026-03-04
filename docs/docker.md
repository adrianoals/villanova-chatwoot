# Comandos Docker — VillaNova

## Rede compartilhada (criar uma vez)
```bash
docker network create villanova-net
```

## Chatwoot
```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot

# Subir
docker compose up -d

# Parar
docker compose down

# Logs
docker compose logs -f rails
docker compose logs -f sidekiq

# Status
docker compose ps

# Backup banco
docker compose exec postgres pg_dump -U postgres chatwoot_production > backup.sql

# Reset banco (CUIDADO: apaga tudo)
docker compose run --rm rails bundle exec rails db:reset
docker compose run --rm rails bundle exec rails db:migrate db:seed

# Console Rails
docker compose exec rails bundle exec rails console
```

## Evolution API
```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api

# Subir
docker compose up -d

# Parar
docker compose down

# Logs
docker compose logs -f evolution-api

# Status
docker compose ps
```

## Subir tudo (ordem importa)
```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot && docker compose up -d
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api && docker compose up -d
```

## Parar tudo
```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api && docker compose down
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot && docker compose down
```

## Verificar conectividade entre containers
```bash
# Evolution API -> Chatwoot
docker exec evolution_api wget -qO- --timeout=5 http://chatwoot-rails-1:3000/

# Chatwoot -> Evolution API
docker exec chatwoot-sidekiq-1 wget -qO- --timeout=5 http://evolution_api:8080/
```

## Verificar WhatsApp
```bash
# Status da conexão
curl -s http://localhost:8080/instance/connectionState/villanova-whatsapp \
  -H "apikey: VillaNova_Evo_2026_TestKey"

# Config do Chatwoot na instância
curl -s http://localhost:8080/chatwoot/find/villanova-whatsapp \
  -H "apikey: VillaNova_Evo_2026_TestKey"

# Enviar mensagem de teste
curl -X POST http://localhost:8080/message/sendText/villanova-whatsapp \
  -H "apikey: VillaNova_Evo_2026_TestKey" \
  -H "Content-Type: application/json" \
  -d '{"number": "5511999999999", "text": "Teste!"}'
```
