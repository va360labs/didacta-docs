# Environment variables

A **one-to-one** reference for every Didacta environment variable, grouped by area. The commented template lives in the repository's [`.env.example`](https://github.com/va360labs/didacta-io/blob/main/.env.example).

**"Component" legend**: `api` = NestJS (`apps/api`) · `web` = Next.js (`apps/web`) · `worker` = BullMQ workers inside the API process · `compose` = read only by Docker Compose · `build` = image build only · `tests` = test/E2E suites.

## Operational summary

**Minimum to start** (the app fails or degrades badly without them):

1. `ADMIN_DATABASE_URL` (or `DATABASE_URL` as a fallback — see § Database)
2. `REDIS_URL`
3. `AUTH_SECRET` — the only one that causes a **hard crash at startup** if missing or shorter than 32 characters.

With `docker-compose.alpha.yml` the first two are composed automatically: **only `AUTH_SECRET` is required**.

**Strongly recommended in production:**

- `WEB_PUBLIC_URL` — without it, every email carries links to `localhost`.
- `TENANT_SETTINGS_ENC_KEY` — set it **before** configuring OIDC SSO, SCIM, per-tenant SMTP, Stripe or Zoom S2S.
- `METRICS_TOKEN` — if `/metrics` is reachable from the internet.
- `AUDIT_REPORT_HMAC_KEY` — the embedded fallback is public (it is in the repository).
- `WEB_PUBLIC_ALLOWED_HOSTS` — if SSO is enabled.
- `POSTGRES_PASSWORD` and `MINIO_ROOT_PASSWORD` set to something other than the `didacta_dev` default.

---

## 1. Core / application

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `NODE_ENV` | No | `production` | api, web, compose | Execution mode. Controls the Pino logger level (`info` in production, `debug` elsewhere). With `test`, BullMQ workers and telemetry are disabled. `DIDACTA_DEV_BYPASS` is ignored when this is `production`. |
| `API_PORT` | No | `4000` | api, compose | Port of the NestJS API. In `docker-compose.alpha.yml` the **internal** port is pinned to 4000 and this variable only controls the host mapping. |
| `WEB_PORT` | No | `3000` | web, compose | Next.js port. Same as `API_PORT`: in the alpha compose file it only maps the host port. |
| `DIDACTA_IMAGE_TAG` | No | the repository's recommended version | compose | Tag of the `didactaio/community` image the compose file brings up. Always pin a specific version. |
| `DIDACTA_CORE_VERSION` | No | derived from the build / `DIDACTA_IMAGE_TAG` | api, compose | Source of truth for the core version: consumed by `/healthz`, by marketplace module compatibility validation and by telemetry. Not normally touched. |
| `RLS_ENFORCEMENT` | No | `on` | api | Row-Level Security enforcement at runtime: `off` \| `warn` \| `on`. In `warn`/`on`, every query with a tenant context travels with `set_config('app.current_tenant_id')`; queries without a context are logged at warning (`warn`) or error (`on`) level. Isolation is real because `DATABASE_URL` connects with the `didacta_app` role (without `BYPASSRLS`) by default — see § Database. |
| `LOG_DB_QUERIES` | No | — | api | With `true`, Prisma logs every SQL query. Very verbose; debugging only. |
| `DIDACTA_CNAME_TARGET` | No | `cname.didacta.io` | api | CNAME target shown to the admin by the custom domains UI. |

## 2. Public URLs and API ↔ Web routing

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `WEB_PUBLIC_URL` | Strongly recommended in prod | cascade (see note) | api, worker | Public base URL of the frontend. It builds every absolute link in emails (password resets, invitations, reminders, digests…). Cascade: this variable → the request's `X-Forwarded-*` headers → `http://localhost:3000`. |
| `WEB_PUBLIC_ALLOWED_HOSTS` | No | — | api | CSV allowlist of hosts for redirects that carry session tokens (SSO / WP-SSO callbacks). Without it, the request host is never used to redirect tokens — it protects against open redirects. Format: `domain1.com,domain2.com`. |
| `PUBLIC_API_URL` | No | falls back to the web base | api | Public base URL of the API. Used by branded emails (assets/logos served from the API) and as a fallback for the OIDC `redirect_uri` and the SAML URLs. |
| `PUBLIC_BASE_URL` | No | `''` (relative path) | api | Absolute base that local storage prefixes to `/api/v1/storage/file/…` so file URLs are absolute. |
| `WEB_BASE_URL` | No | `http://localhost:3000` | api | Base URL injected into the `ModuleContext` of marketplace modules. Different from `WEB_PUBLIC_URL` (which belongs to the core). |
| `API_INTERNAL_URL` | No | `http://localhost:4000` | web | Internal API URL used by the Next.js `rewrites`, server-side fetch and middleware. In the browser the same origin is always used — no CORS. |
| `NEXTAUTH_URL` | No | — | api | A NextAuth legacy; only the second fallback for the web base in OIDC/SAML. Do not use it in new installations. |

!!! note "The frontend exposes no variables to the browser"
    Didacta **uses no `NEXT_PUBLIC_*` variables of its own** in the client bundle: all web configuration is server-side and the browser always talks to its own origin thanks to the Next.js rewrites.

## 3. Database (PostgreSQL)

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `ADMIN_DATABASE_URL` | **Yes** (or `DATABASE_URL` as a fallback) | — | api, worker, compose | The **administration** connection (superuser/owner): `postgresql://user:pass@host:port/db?schema=public`. The entrypoint uses it ONLY for migrations + `rls.sql` + `grants.sql` — never to serve traffic. In the alpha compose file it is built automatically from `POSTGRES_*`. |
| `DATABASE_URL` | No — **leave it empty** | derived from `ADMIN_DATABASE_URL` | api, worker, compose | The app's **runtime** connection. If you leave it empty (the normal case), the entrypoint derives it by swapping the username/password of `ADMIN_DATABASE_URL` for the `didacta_app` role (without `BYPASSRLS` — real RLS isolation). If you set it explicitly, it is honoured as is (upgrade path: existing installations that only had this variable pointing at the superuser still start, with a logged degradation). |
| `POSTGRES_USER` | No | `didacta` | compose | User of the Postgres container (the one in `ADMIN_DATABASE_URL`). |
| `POSTGRES_PASSWORD` | No | `didacta_dev` | compose | Password of the Postgres container. **Change it in any non-local deployment** (the compose file publishes Postgres on `127.0.0.1` only precisely because of this default). |
| `POSTGRES_DB` | No | `didacta` | compose | Database name. |
| `POSTGRES_PORT` | No | `5432` | compose | Host port mapped to Postgres (published on loopback only). |
| `POSTGRES_APP_PASSWORD` | No | auto-generated and persisted in the data volume | compose | Password of the `didacta_app` runtime role. If you leave it empty (the normal case), the entrypoint generates it once and persists it — you never need to know it. Set it only if you want to manage it from your secrets manager. |

## 4. Redis, queues and workers

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `REDIS_URL` | **Yes** (in practice) | — | api, worker, compose | Redis connection (`redis://redis:6379`). Without it, every BullMQ worker degrades silently (outbox, digests, broadcasts, GDPR purges, reminders…), along with distributed rate limiting, realtime messaging and SSE notifications. |
| `REDIS_PORT` | No | `6379` | compose | Host port mapped to Redis (loopback only: Redis runs without authentication). |
| `COMMUNITY_DIGEST_CRON` | No | `0 9 * * 1` | worker | Cron for the weekly community digest (Mondays at 09:00 UTC). |
| `COMMUNITY_BROADCAST_BATCH_SIZE` | No | `5` | worker | Recipients per batch in the community broadcast worker. |
| `COMMUNITY_BROADCAST_INTERVAL_MS` | No | `10000` | worker | Milliseconds between broadcast batches (anti-spam throttling for SMTP). |
| `LESSON_UNLOCK_CRON` | No | `*/10 * * * *` | worker | Cron for the unlocked-lesson notifier (drip content). |
| `PAYMENT_SUBSCRIBERS_SYNC_CRON` | No | `*/15 * * * *` | worker | Cron that syncs subscribers from the payment connections. |
| `REFERRALS_APPROVAL_CRON` | No | `30 * * * *` | worker | Cron for automatic referral approval. |
| `SURVEYS_REMINDER_CRON` | No | `*/15 * * * *` | worker | Cron for the post-class survey reminder. |
| `SURVEYS_REMINDER_DELAY_HOURS` | No | `24` | worker | Hours after the class before the survey reminder is sent. |
| `SURVEYS_HASH_SECRET` | No | falls back to `AUTH_SECRET` | api | Secret used to hash identifiers in survey responses (anonymisation). |

## 5. Authentication, sessions and encryption

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `AUTH_SECRET` | **Yes** | — | api, worker | Signing secret for JWTs (access + refresh), enrollment tickets and unsubscribe tokens. **At least 32 characters**: the app throws at startup if it is missing or shorter. Generate it with `openssl rand -base64 32`. Changing it invalidates every session. |
| `AUTH_URL` | No | `https://didacta.local` | api | JWT issuer (`iss`). Also the default base for Stripe checkout success/cancel URLs. |
| `AUTH_SIGNUP_ENABLED` | No | `false` | api, web | Enables public sign-up (`POST /auth/signup` + the `/signup` page). Closed by design: real accounts come in through registration or an admin invitation. Only set it to `true` in dev/E2E stacks. |
| `TENANT_SETTINGS_ENC_KEY` | Critical if you use integrations | — (cascade) | api, compose | AES-256 key, **64 hex characters**, encrypting per-tenant secrets at rest (OIDC client secret, SCIM token, per-tenant SMTP, Stripe keys, Zoom S2S, Telegram bot). Cascade: this variable → the `${STORAGE_ROOT}/.didacta-secret-key` file (auto-generated, `0600`) → an ephemeral in-memory key (secrets are lost on restart). Generate it with `openssl rand -hex 32`. **Never rotate it without a re-encryption plan.** |
| `TENANT_SETTINGS_ENC_KEY_FILE` | No | `${STORAGE_ROOT}/.didacta-secret-key` | api | Alternative path for the master key file, so the key and the storage can live on different volumes. |
| `DIDACTA_REQUIRE_MFA_ADMIN` | No | `false` | api | Forces MFA on admins across the whole instance. Accepts `true`/`1`/`yes`/`on`. Independent of the per-tenant policy (the `feat:mfa.enforcement` Enterprise capability). |
| `OIDC_REDIRECT_URI` | No | falls back to `PUBLIC_API_URL` | api | The `redirect_uri` sent to the OIDC IdP; it must match the one registered with the provider. |
| `SAML_PUBLIC_API_URL` | No | falls back to `PUBLIC_API_URL` | api | Public API base for the SAML ACS callback. |
| `SAML_SP_ENTITY_ID_BASE` | No | `<API base>/saml` | api | Base of the SAML Service Provider `entityID`. |
| `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN` | No | `100` | api | Requests per minute for authenticated users on the Community plan (Enterprise: 1000, not configurable). |
| `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` | No | `30` | api | Requests per minute for anonymous traffic on Community (Enterprise: 300). Fixed 60 s window. |

## 6. Email / SMTP

The first three work as a set: if any of `SMTP_HOST`, `SMTP_PORT` or `SMTP_FROM` is missing there is no instance-wide SMTP — only tenants with their own SMTP configured in the panel will send email.

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `SMTP_HOST` | No | `mailpit` in the compose file | api, worker, compose | Host of the instance-wide SMTP server. |
| `SMTP_PORT` | No | `1025` (Mailpit) | api, compose | SMTP port. |
| `SMTP_FROM` | No | `Didacta <noreply@didacta.local>` | api, compose | Sender for every email. RFC format: `Name <email@domain>`. |
| `SMTP_USER` | No | `''` | api | SMTP authentication user. Empty = no auth (the Mailpit case). |
| `SMTP_PASS` | No | `''` | api | SMTP password (preferred name; the `SMTP_PASSWORD` alias exists for compatibility). |
| `SMTP_SECURE` | No | decided by nodemailer from the port | api | Forces implicit TLS. Only the literals `'true'`/`'false'` have any effect. |
| `MAILPIT_SMTP_PORT` | No | `1025` | compose | Host port of Mailpit's SMTP server. |
| `MAILPIT_UI_PORT` | No | `8025` | compose | Host port of the Mailpit UI (loopback only: it shows **every** message, password resets included). |

## 7. Object storage (local / S3 / MinIO)

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `STORAGE_DRIVER` | No | `local` in the compose file; auto-detection when unset | api, compose | `local` forces disk; `s3` forces S3 (and **fails at startup** if `S3_ENDPOINT`/`S3_BUCKET`/`S3_ACCESS_KEY`/`S3_SECRET_KEY` are missing). When unset, it tries S3 and falls back to local if the configuration is incomplete. |
| `STORAGE_ROOT` | No | `/app/data/storage` | api, compose | Root of the persistent volume: uploads (logos, certificates, SCORM, Fundae evidence) **and** the `.didacta-secret-key` master key. |
| `S3_ENDPOINT` | Yes, if you use S3 | — | api, compose | S3-compatible endpoint (`http://minio:9000`, AWS, Hetzner…). |
| `S3_BUCKET` | Yes, if you use S3 | — | api | Bucket name. |
| `S3_ACCESS_KEY` | Yes, if you use S3 | — | api | Access key (accepted alias: `S3_ACCESS_KEY_ID`). |
| `S3_SECRET_KEY` | Yes, if you use S3 | — | api | Secret key (accepted alias: `S3_SECRET_ACCESS_KEY`). |
| `S3_REGION` | No | `us-east-1` | api | Bucket region. |
| `S3_FORCE_PATH_STYLE` | No | `true` | api | Path-style URLs (`endpoint/bucket/key`). Required for MinIO. |
| `S3_PRESIGNED_TTL_SECONDS` | No | the SDK default | api | TTL in seconds of the presigned URLs. |
| `MINIO_ROOT_USER` | No | `didacta` | compose | Root user of the MinIO container (the `s3` profile). |
| `MINIO_ROOT_PASSWORD` | No | `didacta_dev` | compose | MinIO root password. Change it outside local development. |
| `MINIO_API_PORT` | No | `9000` | compose | Host port of the MinIO S3 API. |
| `MINIO_CONSOLE_PORT` | No | `9001` | compose | Host port of the MinIO web console. |

## 8. AI — AI Gateway (BYOK)

The canonical AI configuration is **per tenant**, from the admin panel. These variables define the encryption key and the global fallback provider. The names in the `DEFAULT_AI_*` block are built by purpose: `chat` and `embed`.

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `AI_CONFIG_ENCRYPTION_KEY` | Yes, to configure AI per tenant | — | api | AES-256-GCM key encrypting per-tenant AI API keys. **Exactly 64 hex characters** or the module fails to initialise. One key per instance. |
| `DEFAULT_AI_CHAT_PROVIDER` | No | — | api | Global default chat provider, used when a tenant has no configuration of its own. |
| `DEFAULT_AI_CHAT_API_KEY` | No | — | api | API key of the global chat provider. |
| `DEFAULT_AI_CHAT_MODEL` | No | `''` | api | Default chat model. |
| `DEFAULT_AI_CHAT_BASE_URL` | No | — | api | Alternative base URL (proxies, compatible gateways, self-hosted). |
| `DEFAULT_AI_EMBED_PROVIDER` | No | — | api | Global embeddings provider. |
| `DEFAULT_AI_EMBED_API_KEY` | No | — | api | API key of the embeddings provider. |
| `DEFAULT_AI_EMBED_MODEL` | No | `''` | api | Embeddings model. |
| `DEFAULT_AI_EMBED_BASE_URL` | No | — | api | Base URL of the embeddings provider. |

## 9. Zoom (live classes)

Zoom's Server-to-Server credentials **are not environment variables**: they live encrypted per tenant (which is why `TENANT_SETTINGS_ENC_KEY` matters).

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `ZOOM_WEBHOOK_SECRET` | Yes, to receive webhooks | — | api | Zoom secret token used to verify the webhook's HMAC-SHA256 signature. Without it, events are rejected. |
| `ZOOM_REMINDER_CRON` | No | `*/5 * * * *` | worker | Cron for the live class reminder worker. |
| `ZOOM_REMINDER_HOURS_BEFORE` | No | `2` | worker | How many hours ahead the reminder is sent. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | No | `*/10 * * * *` | worker | Cron for attendance reconciliation against the Zoom API. |

## 10. Payments — Stripe (the billing and subscriptions modules)

The canonical configuration lives **per tenant** under Administration → Settings → Payments (`/admin/configuracion`, the "Payments" tab): each academy stores its own secret key and webhook secret, encrypted. These variables are only the instance-wide **global fallback** — a tenant that has not configured its own lands here; if there is nothing here either, that tenant's checkout returns 503 until Stripe is configured somewhere (it never blocks the startup of `mod.billing`/`mod.subscriptions`, which are always registered).

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `STRIPE_SECRET_KEY` | No — instance fallback | — | api | Stripe secret key. A tenant with no credentials of its own in the panel lands here; one account per instance, shared by billing and subscriptions. |
| `STRIPE_WEBHOOK_SECRET` | No | falls back to `SUBSCRIPTIONS_WEBHOOK_SECRET` | api | Secret of the billing webhook (`…/modules/billing/webhook`). |
| `BILLING_SUCCESS_URL_BASE` | No | `<AUTH_URL>/cursos` | api | Base of the success URL for the one-off purchase checkout. |
| `BILLING_CANCEL_URL_BASE` | No | `<AUTH_URL>/cursos` | api | Base of the cancel URL for the one-off checkout. |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | No | falls back to `STRIPE_WEBHOOK_SECRET` | api | Secret of a **dedicated** webhook for subscriptions (keeping it separate is recommended to isolate auditing). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` | No | `<AUTH_URL>/cuenta/suscripciones` | api | Base of the success URL for the subscription checkout. |
| `SUBSCRIPTIONS_CANCEL_URL_BASE` | No | `<AUTH_URL>/cuenta/suscripciones` | api | Base of the cancel URL for the subscription checkout. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | No | `3` | api | Days between the first failed charge and marking the subscription `UNPAID` and pausing the enrollment. |
| `SUBSCRIPTIONS_GRACE_EXPIRATION_CRON` | No | `0 * * * *` | worker | Cron that expires subscriptions whose grace period has run out. |
| `SUBSCRIPTIONS_DAILY_CRON` | No | `0 9 * * *` | worker | Cron for the daily summary and the advance renewal notices. |
| `SUBSCRIPTIONS_DAILY_TZ` | No | `UTC` | worker | IANA time zone of the daily cron (`Europe/Madrid`…). |
| `SUBSCRIPTIONS_RENEWAL_WINDOW_DAYS` | No | `7` | worker | Window, in days, for the notice ahead of renewal/expiry. |

## 11. Member registration / Telegram

The canonical configuration lives **per tenant** under Administration → Settings → Registration (required verifiers, Telegram bot with an encrypted token, approver email). These variables are only the **global fallback** for single-tenant deployments; leave them empty in new installations.

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `TELEGRAM_BOT_TOKEN` | No | — | api | Bot token (global fallback). Requires `TELEGRAM_GROUP_ID` alongside it. If a tenant requires the Telegram verifier and there is no bot, its registration returns a fail-closed 503. |
| `TELEGRAM_GROUP_ID` | No | — | api | Numeric group ID (prefixed with `-100…` in supergroups). |
| `TELEGRAM_BOT_USERNAME` | No | — | api, web | Bot username **without the `@`**, needed for the Telegram Login Widget on the registration form. |
| `MEMBER_APPROVAL_EMAIL` | No | — | api | Email of the approver who receives registration requests (global fallback; prefer the per-tenant setting). |
| `MEMBER_PURGE_CRON` | No | `0 3 * * *` | worker | Cron for the GDPR purge of expired requests. |
| `MEMBER_RETENTION_DAYS` | No | `90` | worker | Retention days before request data is purged. |

## 12. Enterprise license and marketplace

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `DIDACTA_LICENSE_KEY` | No | — (Community mode) | api, compose | Enterprise license JWT signed by Didacta, read **once at startup**. Without it: `License: community` and EE capabilities disabled, unless a license was activated from **Administration → License** (the environment wins over the panel). If invalid: HTTP 402/401 on gated endpoints. Changing it through this route requires restarting the container; from the panel it reloads hot. |
| `DIDACTA_DEV_BYPASS` | No | `false` | api, compose | License bypass: enables **every** capability. **Development only** — it is ignored with `NODE_ENV=production` and emits a visible WARN. |
| `MARKETPLACE_PUBLIC_KEYS_DIR` | No | the embedded directory | api | Directory holding the public keys that verify the signature of marketplace module `.zip` packages. |

## 13. Telemetry, registration and observability

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `DIDACTA_TELEMETRY_DISABLED` | No | `false` | api | Disables the anonymous daily heartbeat. Accepts `true`/`1`/`yes`. See [Telemetry](telemetria.md). |
| `DIDACTA_TELEMETRY_URL` | No | falls back to `DIDACTA_REGISTRY_URL` | api | Alternative heartbeat endpoint (tests, corporate proxies). Fixed 5 s timeout. |
| `DIDACTA_REGISTRY_URL` | No | — (registration disabled) | api | Endpoint of the **identified opt-in registration** (a different level from the anonymous heartbeat). |
| `METRICS_TOKEN` | No | — (`/metrics` open) | api | Optional bearer token for the Prometheus `/metrics` endpoint. If it is defined, it is required. |
| `AUDIT_REPORT_HMAC_KEY` | No | embedded fallback key | api | HMAC-SHA256 master key signing exported audit reports (verifiable offline with `tools/audit-report-verify.mjs`). In production it **must** be overridden: the fallback is public. |
| `WEBHOOKS_TIMEOUT_MS` | No | `5000` | api, worker | Timeout of the POST to an outgoing webhook endpoint. |
| `WEBHOOKS_COMMUNITY_MAX_ENDPOINTS` | No | `1` | api | Maximum webhook endpoints per tenant on Community (Enterprise: 20). |
| `WEBHOOKS_COMMUNITY_MAX_EVENTS` | No | `3` | api | Maximum event types per endpoint on Community (Enterprise: unlimited). |
| `WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS` | No | `20` | api | Maximum endpoints with the `feat:api.webhooks.high_throughput` capability. |

## 14. Development seed (`BOOTSTRAP_*`)

The **real setup wizard** (`/setup`) reads no environment variables at all: the first start is configured through the web interface. These variables belong exclusively to the programmatic development seed (`pnpm db:seed`).

| Name | Required | Default | Component | Description |
|---|---|---|---|---|
| `BOOTSTRAP_TENANT_SLUG` | No | `demo` | seed | Slug of the seeded tenant. |
| `BOOTSTRAP_TENANT_NAME` | No | `Demo` | seed | Display name of the tenant. |
| `BOOTSTRAP_EMAIL` | No | `admin@example.com` | seed | Email of the initial admin. |
| `BOOTSTRAP_NAME` | No | `Admin` | seed | Name of the initial admin. |
| `BOOTSTRAP_PASSWORD` | Yes, for the seed | — | seed | Admin password. Deliberately without a default, so no known credentials are ever created. |
| `BOOTSTRAP_DOMAINS` | No | `localhost,127.0.0.1` | seed | CSV domains associated with the tenant (tenant resolution by host). |

## 15. Tests and E2E (contributors)

| Name | Default | Description |
|---|---|---|
| `TEST_DATABASE_URL` | — | Database for the integration tests (the ephemeral Postgres from `docker-compose.test.yml`, port 5433). |
| `E2E_BASE_URL` | `http://localhost:3010` | Playwright's `baseURL`. |
| `E2E_API_URL` | `http://localhost:3000` | API base for the E2E HTTP helpers. |
| `E2E_ADMIN_EMAIL` / `E2E_ADMIN_PASSWORD` | — | Credentials of the seeded admin the E2E suite signs in with. |
| `E2E_TENANT_SLUG` | `demo` | Tenant the E2E suite runs against. |
| `E2E_AUTH_SECRET` | falls back to `AUTH_SECRET` | Signs Telegram tickets in the registration E2E tests. |
| `E2E_TELEGRAM_BOT_TOKEN` | falls back to `TELEGRAM_BOT_TOKEN` | Signs the Telegram Login Widget payload in E2E. |
| `E2E_STRIPE_WEBHOOK_SECRET` / `E2E_ZOOM_WEBHOOK_SECRET` | — | Sign simulated webhooks in the payments/Zoom E2E tests. |
| `PROD_SMOKE` | — | Switch for the smoke tests against production (only with the exact value `1`). |
| `PROD_TEST_EMAIL` / `PROD_TEST_PASSWORD` / `PROD_TENANT_SLUG` / `PROD_COURSE_SLUG` / `PROD_EXPECTED_COURSE` | — | Parameters of the production smoke test. |

## 16. Image build variables

Only relevant if you build your own image with `docker build`:

| Name | Default | Description |
|---|---|---|
| `DIDACTA_VERSION` | the repository version | Dockerfile `ARG`; it materialises as `DIDACTA_CORE_VERSION`. |
| `SKIP_TYPE_CHECK` | `0` | With `1`, the Next.js build skips typecheck and ESLint (types are validated in CI). |
| `NEXT_TELEMETRY_DISABLED` | `1` | Next.js's own telemetry, disabled in the official image. |
| `HUSKY` | `0` | Disables Git hooks inside the image. |
