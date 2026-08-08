# Configuration

Didacta is configured in **two complementary layers**:

## 1. Environment variables (the installation)

They define the instance infrastructure: database, Redis, storage, instance-wide SMTP, license, telemetry… They are set in `.env` (Docker Compose) or in the container environment, and apply to the **whole installation**.

→ [One-to-one reference for every variable](variables-de-entorno.md)

Only 3 are required (`ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`); the rest have sensible defaults.

## 2. Per-tenant settings (admin panel)

Everything specific to an organization lives in the database and is managed from **Administration** in the web app — never in code and never in environment variables:

| Setting | Where |
| --- | --- |
| Branding: logo, colors, sign-in copy | Administration → Branding |
| The tenant's own SMTP server | Settings → Notifications |
| AI provider and API key (BYOK) | Administration → AI providers |
| Zoom Server-to-Server credentials | Administration → Integrations & API |
| Registration verifiers, Telegram bot | Administration → Settings → Registration |
| Opt-in registration with the Didacta team | Administration → Settings → Registration |

Per-tenant secrets (OIDC client secrets, tokens, Stripe keys, Telegram bot…) are **encrypted at rest** with the instance master key (`TENANT_SETTINGS_ENC_KEY`, or the key auto-generated in the data volume).

!!! warning "Set the encryption key before configuring integrations"
    If you neither define `TENANT_SETTINGS_ENC_KEY` nor persist the data volume, the encryption key will be ephemeral and per-tenant secrets will be lost on restart. Details in [Environment variables → Authentication and encryption](variables-de-entorno.md#5-authentication-sessions-and-encryption).

## Guides by area

- [Database](base-de-datos.md) — PostgreSQL, pgvector, RLS and migrations.
- [Storage](almacenamiento.md) — local disk, MinIO or S3.
- [Email](email.md) — instance-wide and per-tenant SMTP.
- [AI (BYOK)](ia.md) — the multi-provider AI Gateway.
- [Virtual classroom (Zoom)](zoom.md) — credentials and webhooks.
- [Whitelabel branding](branding.md) — per-tenant branding and domains.
- [Telemetry](telemetria.md) — what is sent and how to disable it.
