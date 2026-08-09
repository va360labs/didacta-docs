# Install with docker run

For operators who already have **managed Postgres 16 and Redis 7** (RDS, Cloud SQL, a self-managed Postgres…) and only want to run the application container.

## Prerequisites

- Postgres 16 with the **pgvector** extension installed and an empty schema. The application applies the Prisma migrations at startup.
- Redis 7 reachable from the container.
- The 3 required variables: `ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET` (32+ characters).

## Running it

```bash
docker pull didactaio/community:<version>

# Volume for uploads + the auto-generated encryption key + the
# auto-generated password of the didacta_app runtime role.
docker volume create didacta_data

docker run -d \
  --name didacta-app \
  -p 3000:3000 \
  -p 4000:4000 \
  -v didacta_data:/app/data \
  -e ADMIN_DATABASE_URL='postgresql://USER:PASS@HOST:5432/didacta?schema=public' \
  -e REDIS_URL='redis://HOST:6379' \
  -e AUTH_SECRET='any-random-string-of-32+-characters' \
  -e STORAGE_DRIVER=local \
  -e STORAGE_ROOT=/app/data/storage \
  -e NODE_ENV=production \
  --restart unless-stopped \
  didactaio/community:<version>
```

`ADMIN_DATABASE_URL` is the superuser/owner account you already had: the entrypoint uses it only for migrations + RLS + grants. **Do not set `DATABASE_URL`** — the entrypoint derives it on its own for the `didacta_app` runtime role (no `BYPASSRLS`, real RLS isolation), with a password generated and persisted in `didacta_data` on first start. Details in [Database](../configuracion/base-de-datos.md).

!!! warning "In practice, the `didacta_data` volume is not optional"
    It holds the uploaded files — courses, certificates, evidence — **and** an encryption key generated on first start for secrets at rest. Without that volume mounted, everything is lost when the container is recreated.

    If you use S3 instead of local disk (drop `STORAGE_DRIVER`/`STORAGE_ROOT` and add `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`), **keep the volume mounted anyway** for the encryption key.

## Verifying

```bash
docker logs -f didacta-app                  # bootstrap + Prisma migrations
curl -fsS http://localhost:4000/healthz     # must return 200
```

Then open `http://localhost:3000` and follow the [setup wizard](setup-wizard.md).

## Useful optional variables

| Variable | What it is for |
| --- | --- |
| `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | S3-compatible storage (MinIO, AWS, Hetzner…). |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | Transactional email. Without these, emails are written to the logs but not sent. |
| `DIDACTA_LICENSE_KEY` | Signed JWT that activates Enterprise capabilities. Without it, pure Community mode. |
| `METRICS_TOKEN` | Bearer token protecting `/metrics` (Prometheus). Empty = public endpoint. |

The one-to-one reference for every variable is in [Environment variables](../configuracion/variables-de-entorno.md).
