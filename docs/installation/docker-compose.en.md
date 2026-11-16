---
render_macros: true
---

# Install with Docker Compose

The recommended path. When you are done you will have the full platform — web + API + Postgres + Redis + a test mailbox — running on your machine or server.

The stack is defined by `docker-compose.alpha.yml`, the repository's canonical self-host experience.

## Install with one command

If you just want to see it running, the installer does the five steps below for you (it needs Docker with Compose v2 and `curl` or `wget`):

```bash
curl -fsSL https://raw.githubusercontent.com/va360labs/didacta-io/main/install.sh | bash
```

It creates the `./didacta` folder (change it with `DIDACTA_DIR`), downloads the compose file, **generates `AUTH_SECRET`** and stores it in `.env` (if there is one already, it does not touch it), pins `DIDACTA_IMAGE_TAG` to the installer's version, starts the stack, waits for `/healthz` to answer and finishes by printing the **setup wizard link** (`http://localhost:3000/setup?token=…`, valid once), so you do not have to hunt for the token in the logs. If you prefer, read it before running it: `curl -fsSL <url> -o install.sh && less install.sh && bash install.sh`.

Optional installer variables: `DIDACTA_DIR`, `DIDACTA_IMAGE_TAG`, `DIDACTA_COMPOSE_FILE` (a local compose file), `WEB_PORT`, `API_PORT`, `MAILPIT_UI_PORT` and — to **coexist with another installation on the same machine** — `DIDACTA_PROJECT` (compose project name), `DIDACTA_COMPOSE_EXTRA` (an override file) and `DIDACTA_NAME_SUFFIX` (suffix for container names, for example `-2`: `container_name`s are global to the host and the project name does not isolate them).

## Manual installation

```bash
# 1. Download the compose file and the environment template
git clone https://github.com/va360labs/didacta-io.git
cd didacta-io

# 2. Configure the environment. With compose, only AUTH_SECRET is required.
cp .env.example .env
# Edit .env and fill AUTH_SECRET with a long, random secret:
#   openssl rand -base64 32

# 3. Pin the image version (published tags are on GitHub Container Registry)
echo "DIDACTA_IMAGE_TAG={{ didacta_version }}" >> .env

# 4. Start the stack
docker compose -f docker-compose.alpha.yml up -d

# 5. Check that everything is healthy (~60-90 s the first time)
docker compose -f docker-compose.alpha.yml ps
```

On first start the app container automatically applies the **versioned migrations** (`prisma migrate deploy`), the **RLS policies** and the **idempotent system seed**. There is nothing to run by hand.

## Installation URLs

| URL | What it is |
| --- | --- |
| `http://localhost:3000` | Web app (the first time it redirects to the [setup wizard](setup-wizard.md)) |
| `http://localhost:4000/api/docs` | API Swagger |
| `http://localhost:4000/healthz` | Health probe |
| `http://localhost:8025` | Mailpit — the mailbox where test emails land |

## First sign-in

1. Open the wizard link: `http://localhost:3000/setup?token=…`. The token appears in the container logs (`docker compose -f docker-compose.alpha.yml logs didacta | grep "Setup token"`) or `install.sh` prints it; without it the wizard will not let you finish the sign-up. There you create the organization (tenant) and the first administrator account.
2. Sign in with that account and configure your brand under **Administration → Branding** (logo, colors, sign-in screen copy).
3. Outgoing mail points to the Mailpit test mailbox by default (`http://localhost:8025`). For production, configure your real SMTP server under **Administration → Settings → Notifications**.

## Persistence: volumes

The compose file declares four named volumes that survive `down`/`up` and restarts:

| Volume | What it holds |
| --- | --- |
| `postgres_data` | The entire database. |
| `redis_data` | The persistent queue (`appendonly yes`) — outbox + jobs. |
| `didacta_data` (`/app/data`) | The application's local storage (`/app/data/storage`: course uploads, certificates and evidence), **the auto-generated encryption key** for data at rest (`/app/data/secrets`, **outside** the tree served over HTTP) and the auto-generated password of the `didacta_app` role. |
| `minio_data` | Only with the `s3` profile. MinIO buckets. |

!!! danger "`down -v` deletes your data"
    `docker compose down -v` removes the volumes and therefore the database and the files. To stop without deleting anything, use `docker compose down` **without** `-v`.

## Passing more variables to the container

The compose file's `didacta` service **only receives the variables its `environment` block declares** (there is no `env_file`): `NODE_ENV`, `ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`, `TENANT_SETTINGS_ENC_KEY`, `AUDIT_REPORT_HMAC_KEY`, `METRICS_TOKEN`, `STORAGE_*`, `SMTP_FROM`, `DIDACTA_LICENSE_KEY` and `DIDACTA_DEV_BYPASS`, among others. A variable you write in `.env` that is not there **does not reach the application** (which is why SMTP always points to Mailpit, and why the `S3_*` ones have to be uncommented). To add others (`WEB_PUBLIC_URL`, `SMTP_HOST`/`SMTP_PORT`, `STRIPE_*`, `SUPPORT_OPERATOR_ALERT_EMAIL`…), create an override file and pass it with `-f`:

```yaml
# docker-compose.override.yml
services:
  didacta:
    environment:
      WEB_PUBLIC_URL: https://campus.example.com
      SMTP_HOST: smtp.your-provider.com
      SMTP_PORT: '587'
```

```bash
docker compose -f docker-compose.alpha.yml -f docker-compose.override.yml up -d
```

(With an explicit `-f`, compose does **not** load `docker-compose.override.yml` on its own.) The platform's SMTP can also be configured per academy from the panel without touching the compose file: see [Email](../configuration/email.md).

!!! warning "Production: two variables worth defining"
    Since the compose file starts with `NODE_ENV=production`: **`/metrics` answers 401 until you define `METRICS_TOKEN`** (only needed if you scrape it) and **the signed audit-log export answers 503 until you define `AUDIT_REPORT_HMAC_KEY`** (only with that Enterprise capability). Generate both with `openssl rand -hex 32`. And set `WEB_PUBLIC_URL`: without it emails carry links to `localhost`.

## File storage

By default, files (cover images, certificates, evidence) are stored on disk in the `didacta_data` volume. To use S3-compatible storage instead:

=== "Local MinIO (testing)"

    ```bash
    docker compose -f docker-compose.alpha.yml --profile s3 up -d
    # MinIO console at http://localhost:9001
    ```

    Then uncomment the `S3_*` variables in the `environment` block of the `didacta` service in the compose file.

=== "External S3 (production)"

    Fill in the `S3_*` variables in `.env` and set `STORAGE_DRIVER=s3`. It works with AWS S3, Hetzner Object Storage or any S3-compatible provider. Details in [Storage](../configuration/storage.md).

## Telemetry

The installation sends an anonymous daily heartbeat (random instance id + version + edition + OS) so live installations can be counted. No PII, and no effect if there is no network access. Disable it with:

```bash
DIDACTA_TELEMETRY_DISABLED=true
```

The payload is documented in detail in [Telemetry](../configuration/telemetry.md).

## Enterprise license

Community works in full without a license. If you have an Enterprise license, put it in `DIDACTA_LICENSE_KEY` and the EE capabilities are unlocked at startup. Without a license those screens stay visible with a notice — never hidden. See [Enterprise](../enterprise/index.md).

## Next step

→ [Setup wizard](setup-wizard.md), or if something does not start, [Troubleshooting](troubleshooting.md).
