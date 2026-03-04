# Setup Local — VillaNova Chatwoot + Evolution API

## Pré-requisitos

- Docker Desktop instalado e rodando
- Portas livres: 3000, 5432, 6379, 8080, 9615

## 1. Criar rede compartilhada

```bash
docker network create villanova-net
```

## 2. Subir o Chatwoot

```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/chatwoot

# Criar .env se não existir (copiar de .env.example ou ver CLAUDE.md para variáveis)
docker compose up -d
```

Aguardar ~60 segundos para o Rails iniciar. Verificar:
```bash
docker compose logs -f rails
# Esperar: "Puma starting... Listening on http://0.0.0.0:3000"
```

Acessar: http://localhost:3000

### Primeiro acesso
- Criar conta de admin
- Anotar o **Access Token** em Profile Settings (necessário para integração)
- Anotar o **Account ID** da URL (`/app/accounts/1/...` → ID = 1)

## 3. Subir a Evolution API

```bash
cd C:/Documentos/Projetos/VillaNova-Chatwoot/evolution-api

# Criar .env se não existir (ver CLAUDE.md para variáveis)
docker compose up -d
```

Verificar:
```bash
curl http://localhost:8080/
# Deve retornar: {"status":200,"message":"Welcome to the Evolution API..."}
```

Acessar:
- API: http://localhost:8080 (Swagger: http://localhost:8080/docs)
- Manager: http://localhost:9615

## 4. Criar instância WhatsApp + Chatwoot

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
    "chatwootConversationPending": false,
    "chatwootAutoCreate": true,
    "chatwootNameInbox": "WhatsApp Villa Nova",
    "chatwootImportContacts": true,
    "chatwootImportMessages": true,
    "chatwootDaysLimitImportMessages": 30,
    "chatwootMergeBrazilContacts": true,
    "chatwootOrganization": "Villa Nova Condominios"
  }'
```

## 5. Conectar WhatsApp (QR Code)

```bash
curl http://localhost:8080/instance/connect/villanova-whatsapp \
  -H "apikey: SUA_API_KEY"
```

Escanear o QR code retornado (base64) com o WhatsApp no celular.

Verificar conexão:
```bash
curl http://localhost:8080/instance/connectionState/villanova-whatsapp \
  -H "apikey: SUA_API_KEY"
# Deve retornar: {"instance":{"instanceName":"villanova-whatsapp","state":"open"}}
```

## 6. Corrigir webhook do inbox no Chatwoot

O `autoCreate` pode gerar o webhook com URL errada. Verificar e corrigir:

```bash
# Verificar
curl http://localhost:3000/api/v1/accounts/1/inboxes \
  -H "api_access_token: SEU_CHATWOOT_TOKEN"

# Corrigir se necessário
curl -X PATCH http://localhost:3000/api/v1/accounts/1/inboxes/1 \
  -H "api_access_token: SEU_CHATWOOT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"channel": {"webhook_url": "http://evolution_api:8080/chatwoot/webhook/villanova-whatsapp"}}'
```

## 7. Testar

Enviar mensagem de teste:
```bash
curl -X POST http://localhost:8080/message/sendText/villanova-whatsapp \
  -H "apikey: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"number": "5511999999999", "text": "Teste Villa Nova!"}'
```

A mensagem deve aparecer no Chatwoot em http://localhost:3000.

## Problemas conhecidos

- **"Falha ao enviar" no Chatwoot:** Mensagens são entregues no WhatsApp mas status visual fica como falha. Bug da Evolution API com URLs de container (`ENOTFOUND host`). Deve resolver em produção com domínios reais.
- **Evolution Manager não sobe:** Bug na imagem oficial do nginx. Resolvido com `manager-nginx.conf` montado como volume.
