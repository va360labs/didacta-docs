---
render_macros: true
---

# Install with Docker Compose

The recommended path. When you are done you will have the full platform — web + API + Postgres + Redis + a test mailbox — running on your machine or server.

The stack is defined by `docker-compose.alpha.yml`, the repository's canonical self-host experience.

## Installation

```bash
# 1. Download the compose file and the environment template
git clone https://github.com/va360labs/didacta-io.git
cd didacta-io

# 2. Configure the environment. With compose, only AUTH_SECRET is required.
cp .env.example .env
# Edit .env and fill AUTH_SECRET with a long, random secret:
#   openssl rand -base64 32

# 3. Pin the image version (published tags are on Docker Hub)
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

1. Open `http://localhost:3000`. The first time it takes you to the **setup wizard** (`/setup`), where you create the organization (tenant) and the first administrator account.
2. Sign in with that account and configure your brand under **Administration → Branding** (logo, colors, sign-in screen copy).
3. Outgoing mail points to the Mailpit test mailbox by default (`http://localhost:8025`). For production, configure your real SMTP server under **Administration → Settings → Notifications**.

## Persistence: volumes

The compose file declares four named volumes that survive `down`/`up` and restarts:

| Volume | What it holds |
| --- | --- |
| `postgres_data` | The entire database. |
| `redis_data` | The persistent queue (`appendonly yes`) — outbox + jobs. |
| `didacta_data` | The application's local storage: course uploads, certificates and evidence, **plus the auto-generated encryption key** for data at rest. |
| `minio_data` | Only with the `s3` profile. MinIO buckets. |

!!! danger "`down -v` deletes your data"
    `docker compose down -v` removes the volumes and therefore the database and the files. To stop without deleting anything, use `docker compose down` **without** `-v`.

## File storage

By default, files (cover images, certificates, evidence) are stored on disk in the `didacta_data` volume. To use S3-compatible storage instead:

=== "Local MinIO (testing)"

    ```bash
    docker compose -f docker-compose.alpha.yml --profile s3 up -d
    # MinIO console at http://localhost:9001
    ```

    Then uncomment the `S3_*` variables in the `environment` block of the `didacta` service in the compose file.

=== "External S3 (production)"

    Fill in the `S3_*` variables in `.env` and set `STORAGE_DRIVER=s3`. It works with AWS S3, Hetzner Object Storage or any S3-compatible provider. Details in [Storage](../configuracion/almacenamiento.md).

## Telemetry

The installation sends an anonymous daily heartbeat (random instance id + version + edition + OS) so live installations can be counted. No PII, and no effect if there is no network access. Disable it with:

```bash
DIDACTA_TELEMETRY_DISABLED=true
```

The payload is documented in detail in [Telemetry](../configuracion/telemetria.md).

## Enterprise license

Community works in full without a license. If you have an Enterprise license, put it in `DIDACTA_LICENSE_KEY` and the EE capabilities are unlocked at startup. Without a license those screens stay visible with a notice — never hidden. See [Enterprise](../enterprise/index.md).

## Next step

→ [Setup wizard](setup-wizard.md), or if something does not start, [Troubleshooting](solucion-de-problemas.md).
