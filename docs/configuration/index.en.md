# Configuration

Didacta is configured in **two complementary layers**:

## 1. Environment variables (the installation)

They define the instance infrastructure: database, Redis, storage, instance-wide SMTP, license, telemetry… They are set in `.env` (Docker Compose) or in the container environment, and apply to the **whole installation**.

→ [One-to-one reference for every variable](environment-variables.md)

Only 3 are required (`ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`); the rest have sensible defaults.

## 2. Per-tenant settings (admin panel)

Everything specific to an organization lives in the database and is managed from **Administration** in the web app — never in code and never in environment variables:

| Setting | Where |
| --- | --- |
| Academy language (es, en, pt-BR) | Administration → Settings → General |
| Branding: logo, colors, sign-in copy | Administration → Branding |
| The academy's public page, own domains | [Branding](branding.md) |
| Simple or full mode | [Simple mode](simple-mode.md) |
| The tenant's own SMTP server and "Reply-to email" | Administration → Settings → Notifications |
| Email templates | Administration → Emails |
| AI provider and API key (BYOK) | Administration → AI providers |
| Zoom Server-to-Server credentials | Administration, «Integrations & API» group |
| Registration verifiers, Telegram bot | Administration → Settings → Registration |
| Opt-in registration with the Didacta team | Administration → Settings → Registration |

Per-tenant secrets (OIDC client secrets, tokens, Stripe keys, Telegram bot…) are **encrypted at rest** with the instance master key (`TENANT_SETTINGS_ENC_KEY`, or the key auto-generated in `/app/data/secrets`, inside the data volume but outside the storage that is served over HTTP).

!!! warning "Set the encryption key before configuring integrations"
    If you neither define `TENANT_SETTINGS_ENC_KEY` nor persist the data volume, in production the API **does not start** (and outside production the key would be ephemeral and per-tenant secrets would be lost on restart). Back up the auto-generated key. Details in [Environment variables → Authentication and encryption](environment-variables.md#5-authentication-sessions-and-encryption).

## Guides by area

- [Database](database.md) — PostgreSQL, pgvector, RLS and migrations.
- [Storage](storage.md) — local disk, MinIO or S3.
- [Email](email.md) — instance-wide and per-tenant SMTP.
- [AI (BYOK)](ai.md) — the multi-provider AI Gateway.
- [Virtual classroom (Zoom)](zoom.md) — credentials and webhooks.
- [Whitelabel branding](branding.md) — per-tenant branding, public page and domains.
- [Languages](languages.md) — Spanish, English and Brazilian Portuguese.
- [Simple mode](simple-mode.md) — the short panel to go from zero to selling.
- [Telemetry](telemetry.md) — what is sent and how to disable it.
