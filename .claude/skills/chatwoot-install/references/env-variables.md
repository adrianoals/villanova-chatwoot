# Chatwoot — Environment Variables Reference

Complete list of environment variables for Chatwoot self-hosted.

## Essential (Required)

| Variable | Default | Description |
|----------|---------|-------------|
| `SECRET_KEY_BASE` | — | 64-char hex for cookie signing. Generate with `openssl rand -hex 64` |
| `FRONTEND_URL` | `http://0.0.0.0:3000` | Public URL of the application |
| `POSTGRES_HOST` | `postgres` | PostgreSQL hostname (use `postgres` for Docker) |
| `POSTGRES_USERNAME` | `postgres` | Database user |
| `POSTGRES_PASSWORD` | — | Database password |
| `POSTGRES_DATABASE` | `chatwoot_production` | Database name |
| `REDIS_URL` | `redis://redis:6379` | Redis connection URL |
| `REDIS_PASSWORD` | — | Redis password |

## Application

| Variable | Default | Description |
|----------|---------|-------------|
| `DEFAULT_LOCALE` | `en` | Default language (`pt_BR` for Portuguese) |
| `ENABLE_ACCOUNT_SIGNUP` | `false` | Allow new account signups |
| `FORCE_SSL` | `false` | Force HTTPS redirects |
| `RAILS_ENV` | `development` | Set to `production` for production |
| `RAILS_MAX_THREADS` | `5` | Max threads per Rails process |
| `RAILS_LOG_TO_STDOUT` | `true` | Log to stdout (useful for Docker) |
| `LOG_LEVEL` | `info` | Log level: debug, info, warn, error |

## Encryption (for 2FA/MFA)

| Variable | Default | Description |
|----------|---------|-------------|
| `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY` | — | Primary encryption key |
| `ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY` | — | Deterministic encryption key |
| `ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT` | — | Key derivation salt |

## Email (SMTP)

| Variable | Default | Description |
|----------|---------|-------------|
| `MAILER_SENDER_EMAIL` | `Chatwoot <accounts@chatwoot.com>` | From address |
| `SMTP_ADDRESS` | — | SMTP server (e.g., `smtp.gmail.com`) |
| `SMTP_PORT` | `1025` | SMTP port (587 for TLS, 465 for SSL) |
| `SMTP_USERNAME` | — | SMTP username |
| `SMTP_PASSWORD` | — | SMTP password |
| `SMTP_AUTHENTICATION` | — | Auth method: `plain`, `login`, `cram_md5` |
| `SMTP_ENABLE_STARTTLS_AUTO` | `true` | Enable STARTTLS |
| `SMTP_DOMAIN` | `chatwoot.com` | SMTP HELO domain |

## Storage

| Variable | Default | Description |
|----------|---------|-------------|
| `ACTIVE_STORAGE_SERVICE` | `local` | Storage backend: `local`, `amazon`, `gcs`, `microsoft` |
| `S3_BUCKET_NAME` | — | AWS S3 bucket name |
| `AWS_ACCESS_KEY_ID` | — | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | — | AWS secret key |
| `AWS_REGION` | — | AWS region |

## Security & Rate Limiting

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_RACK_ATTACK` | — | Enable request throttling |
| `RACK_ATTACK_LIMIT` | `300` | Max requests per period |
| `RACK_ATTACK_ALLOWED_IPS` | — | Trusted IPs (comma-separated) |

## Sidekiq (Background Jobs)

| Variable | Default | Description |
|----------|---------|-------------|
| `SIDEKIQ_CONCURRENCY` | `10` | Number of concurrent workers |

## Integrations

| Variable | Description |
|----------|-------------|
| `FB_APP_ID` | Facebook app ID |
| `FB_APP_SECRET` | Facebook app secret |
| `FB_VERIFY_TOKEN` | Facebook webhook verify token |
| `SLACK_CLIENT_ID` | Slack integration client ID |
| `SLACK_CLIENT_SECRET` | Slack integration client secret |
| `OPENAI_API_KEY` | OpenAI API key for AI features |

## Monitoring (Optional)

| Variable | Description |
|----------|-------------|
| `SENTRY_DSN` | Sentry error tracking URL |
| `NEW_RELIC_LICENSE_KEY` | New Relic monitoring key |
| `SCOUT_KEY` | Scout APM key |
| `DD_TRACE_AGENT_URL` | Datadog trace agent URL |
