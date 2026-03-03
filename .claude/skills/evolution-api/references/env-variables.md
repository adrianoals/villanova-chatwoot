# Evolution API — Environment Variables Reference

Complete list of environment variables for Evolution API v2.

## Table of Contents
1. [Server](#server)
2. [Database (PostgreSQL)](#database)
3. [Redis Cache](#redis-cache)
4. [Authentication](#authentication)
5. [Chatwoot Integration](#chatwoot-integration)
6. [Webhooks](#webhooks)
7. [WhatsApp Session](#whatsapp-session)
8. [QR Code](#qr-code)
9. [Logging](#logging)
10. [CORS](#cors)
11. [S3 Storage](#s3-storage)
12. [Other Integrations](#other-integrations)
13. [Monitoring](#monitoring)

---

## Server

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_NAME` | `evolution` | Server name identifier |
| `SERVER_TYPE` | `http` | Server type |
| `SERVER_PORT` | `8080` | API port |
| `SERVER_URL` | `http://localhost:8080` | Public URL of the API (used for webhooks, QR code, etc.) |

## Database

| Variable | Default | Description |
|----------|---------|-------------|
| `DATABASE_PROVIDER` | `postgresql` | Provider: `postgresql`, `mysql`, `psql_bouncer` |
| `DATABASE_CONNECTION_URI` | — | Full connection URI: `postgresql://user:pass@host:5432/db?schema=evolution_api` |
| `DATABASE_CONNECTION_CLIENT_NAME` | `evolution_exchange` | Client identifier (separates installations sharing a DB) |
| `DATABASE_SAVE_DATA_INSTANCE` | `true` | Save instance data |
| `DATABASE_SAVE_DATA_NEW_MESSAGE` | `true` | Save new messages |
| `DATABASE_SAVE_MESSAGE_UPDATE` | `true` | Save message updates |
| `DATABASE_SAVE_DATA_CONTACTS` | `true` | Save contacts |
| `DATABASE_SAVE_DATA_CHATS` | `true` | Save chats |
| `DATABASE_SAVE_DATA_LABELS` | `true` | Save labels |
| `DATABASE_SAVE_DATA_HISTORIC` | `true` | Save historic data |
| `DATABASE_SAVE_IS_ON_WHATSAPP` | `true` | Cache WhatsApp number checks |
| `DATABASE_SAVE_IS_ON_WHATSAPP_DAYS` | `7` | Days to cache number checks |
| `DATABASE_DELETE_MESSAGE` | `true` | Allow message deletion |

PostgreSQL container variables (used by docker-compose):

| Variable | Description |
|----------|-------------|
| `POSTGRES_DATABASE` | Database name |
| `POSTGRES_USERNAME` | Database user |
| `POSTGRES_PASSWORD` | Database password |

## Redis Cache

| Variable | Default | Description |
|----------|---------|-------------|
| `CACHE_REDIS_ENABLED` | `true` | Enable Redis cache |
| `CACHE_REDIS_URI` | `redis://localhost:6379/6` | Redis connection URI |
| `CACHE_REDIS_TTL` | `604800` | Cache TTL in seconds (7 days) |
| `CACHE_REDIS_PREFIX_KEY` | `evolution` | Key prefix (separates installations) |
| `CACHE_REDIS_SAVE_INSTANCES` | `false` | Save instance info in Redis instead of DB |
| `CACHE_LOCAL_ENABLED` | `false` | Enable local cache (fallback) |

## Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `AUTHENTICATION_API_KEY` | `429683C4C977415CAAFCCE10F7D57E11` | Global API key for all requests |
| `AUTHENTICATION_EXPOSE_IN_FETCH_INSTANCES` | `true` | Expose instance tokens in fetch endpoint |

## Chatwoot Integration

| Variable | Default | Description |
|----------|---------|-------------|
| `CHATWOOT_ENABLED` | `false` | Enable Chatwoot integration globally |
| `CHATWOOT_MESSAGE_READ` | `true` | Sync read status to WhatsApp |
| `CHATWOOT_MESSAGE_DELETE` | `true` | Sync deletions to WhatsApp |
| `CHATWOOT_BOT_CONTACT` | `true` | Create bot contact for QR code and instance updates |
| `CHATWOOT_IMPORT_DATABASE_CONNECTION_URI` | — | Direct DB connection to Chatwoot for message import |
| `CHATWOOT_IMPORT_PLACEHOLDER_MEDIA_MESSAGE` | `true` | Use placeholder for media in imports |

## Webhooks

### Global Webhook

| Variable | Default | Description |
|----------|---------|-------------|
| `WEBHOOK_GLOBAL_ENABLED` | `false` | Enable global webhook (all instances) |
| `WEBHOOK_GLOBAL_URL` | — | Global webhook URL |
| `WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS` | `false` | Separate URL per event type |
| `WEBHOOK_REQUEST_TIMEOUT_MS` | `60000` | Request timeout in ms |
| `WEBHOOK_RETRY_MAX_ATTEMPTS` | `10` | Max retry attempts |
| `WEBHOOK_RETRY_INITIAL_DELAY_SECONDS` | `5` | Initial retry delay |
| `WEBHOOK_RETRY_USE_EXPONENTIAL_BACKOFF` | `true` | Use exponential backoff |
| `WEBHOOK_RETRY_MAX_DELAY_SECONDS` | `300` | Max retry delay |
| `WEBHOOK_RETRY_JITTER_FACTOR` | `0.2` | Jitter factor for retries |
| `WEBHOOK_RETRY_NON_RETRYABLE_STATUS_CODES` | `400,401,403,404,422` | HTTP codes that won't be retried |

### Webhook Events (prefix `WEBHOOK_EVENTS_`)

All boolean. Control which events are sent:

| Variable | Description |
|----------|-------------|
| `WEBHOOK_EVENTS_APPLICATION_STARTUP` | API started |
| `WEBHOOK_EVENTS_QRCODE_UPDATED` | QR code generated/updated |
| `WEBHOOK_EVENTS_MESSAGES_SET` | Messages loaded |
| `WEBHOOK_EVENTS_MESSAGES_UPSERT` | New message received |
| `WEBHOOK_EVENTS_MESSAGES_EDITED` | Message edited |
| `WEBHOOK_EVENTS_MESSAGES_UPDATE` | Message status updated (delivered, read) |
| `WEBHOOK_EVENTS_MESSAGES_DELETE` | Message deleted |
| `WEBHOOK_EVENTS_SEND_MESSAGE` | Message sent |
| `WEBHOOK_EVENTS_SEND_MESSAGE_UPDATE` | Sent message status updated |
| `WEBHOOK_EVENTS_CONTACTS_SET` | Contacts loaded |
| `WEBHOOK_EVENTS_CONTACTS_UPSERT` | Contact created/updated |
| `WEBHOOK_EVENTS_CONTACTS_UPDATE` | Contact updated |
| `WEBHOOK_EVENTS_PRESENCE_UPDATE` | Online/offline/typing status |
| `WEBHOOK_EVENTS_CHATS_SET` | Chats loaded |
| `WEBHOOK_EVENTS_CHATS_UPSERT` | Chat created/updated |
| `WEBHOOK_EVENTS_CHATS_UPDATE` | Chat updated |
| `WEBHOOK_EVENTS_CHATS_DELETE` | Chat deleted |
| `WEBHOOK_EVENTS_GROUPS_UPSERT` | Group created/updated |
| `WEBHOOK_EVENTS_GROUPS_UPDATE` | Group settings updated |
| `WEBHOOK_EVENTS_GROUP_PARTICIPANTS_UPDATE` | Group members changed |
| `WEBHOOK_EVENTS_CONNECTION_UPDATE` | Connection state changed |
| `WEBHOOK_EVENTS_LABELS_EDIT` | Label edited |
| `WEBHOOK_EVENTS_LABELS_ASSOCIATION` | Label associated/removed |
| `WEBHOOK_EVENTS_CALL` | Call received |
| `WEBHOOK_EVENTS_TYPEBOT_START` | Typebot session started |
| `WEBHOOK_EVENTS_TYPEBOT_CHANGE_STATUS` | Typebot status changed |
| `WEBHOOK_EVENTS_REMOVE_INSTANCE` | Instance removed |
| `WEBHOOK_EVENTS_LOGOUT_INSTANCE` | Instance logged out |
| `WEBHOOK_EVENTS_ERRORS` | Error occurred |

## WhatsApp Session

| Variable | Default | Description |
|----------|---------|-------------|
| `CONFIG_SESSION_PHONE_CLIENT` | `Evolution API` | Name shown on phone's linked devices |
| `CONFIG_SESSION_PHONE_NAME` | `Chrome` | Browser name: `Chrome`, `Firefox`, `Edge`, `Opera`, `Safari` |

## QR Code

| Variable | Default | Description |
|----------|---------|-------------|
| `QRCODE_LIMIT` | `30` | Max QR code generation attempts |
| `QRCODE_COLOR` | `#175197` | QR code color (hex) |

## Logging

| Variable | Default | Description |
|----------|---------|-------------|
| `LOG_LEVEL` | `ERROR,WARN,DEBUG,INFO,LOG,VERBOSE,DARK,WEBHOOKS,WEBSOCKET` | Log levels to enable |
| `LOG_COLOR` | `true` | Colorized log output |
| `LOG_BAILEYS` | `error` | Baileys library log level: `fatal`, `error`, `warn`, `info`, `debug`, `trace` |

## Instance Management

| Variable | Default | Description |
|----------|---------|-------------|
| `DEL_INSTANCE` | `false` | Auto-delete disconnected instances. `false` = never, or set minutes (e.g. `5`) |
| `EVENT_EMITTER_MAX_LISTENERS` | `50` | Max event listeners |
| `LANGUAGE` | `en` | API language |

## CORS

| Variable | Default | Description |
|----------|---------|-------------|
| `CORS_ORIGIN` | `*` | Allowed origins |
| `CORS_METHODS` | `GET,POST,PUT,DELETE` | Allowed HTTP methods |
| `CORS_CREDENTIALS` | `true` | Allow credentials |

## WhatsApp Business API (Meta Cloud)

| Variable | Default | Description |
|----------|---------|-------------|
| `WA_BUSINESS_TOKEN_WEBHOOK` | `evolution` | Webhook validation token for Facebook |
| `WA_BUSINESS_URL` | `https://graph.facebook.com` | Meta Graph API URL |
| `WA_BUSINESS_VERSION` | `v20.0` | Graph API version |
| `WA_BUSINESS_LANGUAGE` | `en_US` | Default language |

## S3 Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `S3_ENABLED` | `false` | Enable S3 storage for media |
| `S3_ACCESS_KEY` | — | S3 access key |
| `S3_SECRET_KEY` | — | S3 secret key |
| `S3_BUCKET` | `evolution` | S3 bucket name |
| `S3_PORT` | `443` | S3 port |
| `S3_ENDPOINT` | — | S3 endpoint |
| `S3_REGION` | — | S3 region |
| `S3_USE_SSL` | `true` | Use SSL for S3 |

## Proxy

| Variable | Description |
|----------|-------------|
| `PROXY_HOST` | Proxy host |
| `PROXY_PORT` | Proxy port |
| `PROXY_PROTOCOL` | Proxy protocol |
| `PROXY_USERNAME` | Proxy username |
| `PROXY_PASSWORD` | Proxy password |

## Kafka (since v2.3.4)

| Variable | Default | Description |
|----------|---------|-------------|
| `KAFKA_ENABLED` | `false` | Enable Apache Kafka integration |
| `KAFKA_CLIENT_ID` | `evolution-api` | Kafka client identifier |
| `KAFKA_BROKERS` | `localhost:9092` | Kafka broker(s) |
| `KAFKA_CONNECTION_TIMEOUT` | `3000` | Connection timeout in ms |
| `KAFKA_REQUEST_TIMEOUT` | `30000` | Request timeout in ms |
| `KAFKA_GLOBAL_ENABLED` | `false` | Send all instances to global topics |
| `KAFKA_CONSUMER_GROUP_ID` | `evolution-api-consumers` | Consumer group ID |
| `KAFKA_TOPIC_PREFIX` | `evolution` | Topic prefix |
| `KAFKA_NUM_PARTITIONS` | `1` | Number of partitions |
| `KAFKA_REPLICATION_FACTOR` | `1` | Replication factor |
| `KAFKA_AUTO_CREATE_TOPICS` | `false` | Auto-create topics |
| `KAFKA_SASL_ENABLED` | `false` | Enable SASL authentication |
| `KAFKA_SASL_MECHANISM` | `plain` | SASL mechanism |
| `KAFKA_SASL_USERNAME` | — | SASL username |
| `KAFKA_SASL_PASSWORD` | — | SASL password |
| `KAFKA_SSL_ENABLED` | `false` | Enable SSL |

Plus `KAFKA_EVENTS_*` toggles for each event type.

## Pusher

| Variable | Default | Description |
|----------|---------|-------------|
| `PUSHER_ENABLED` | `false` | Enable Pusher integration |
| `PUSHER_GLOBAL_ENABLED` | `false` | Global Pusher events |
| `PUSHER_GLOBAL_APP_ID` | — | Pusher App ID |
| `PUSHER_GLOBAL_KEY` | — | Pusher Key |
| `PUSHER_GLOBAL_SECRET` | — | Pusher Secret |
| `PUSHER_GLOBAL_CLUSTER` | — | Pusher Cluster |
| `PUSHER_GLOBAL_USE_TLS` | `true` | Use TLS |

Plus `PUSHER_EVENTS_*` toggles for each event type.

## Other Integrations

| Variable | Default | Description |
|----------|---------|-------------|
| `TYPEBOT_ENABLED` | `false` | Enable Typebot |
| `TYPEBOT_API_VERSION` | `latest` | Typebot API version (`old` or `latest`) |
| `OPENAI_ENABLED` | `false` | Enable OpenAI |
| `DIFY_ENABLED` | `false` | Enable Dify |
| `N8N_ENABLED` | `false` | Enable n8n |
| `EVOAI_ENABLED` | `false` | Enable EvoAI |

## Monitoring

| Variable | Default | Description |
|----------|---------|-------------|
| `TELEMETRY_ENABLED` | `true` | Enable telemetry |
| `PROMETHEUS_METRICS` | `false` | Enable Prometheus metrics |
| `METRICS_AUTH_REQUIRED` | `true` | Require auth for /metrics |
| `METRICS_USER` | `prometheus` | Metrics auth user |
| `METRICS_PASSWORD` | — | Metrics auth password |
| `METRICS_ALLOWED_IPS` | `127.0.0.1` | Comma-separated allowed IPs |

## Audio Converter

| Variable | Description |
|----------|-------------|
| `API_AUDIO_CONVERTER` | Audio converter API URL |
| `API_AUDIO_CONVERTER_KEY` | Audio converter API key |

## SSL (Direct, without reverse proxy)

| Variable | Description |
|----------|-------------|
| `SSL_CONF_PRIVKEY` | Path to SSL private key |
| `SSL_CONF_FULLCHAIN` | Path to SSL full chain certificate |
