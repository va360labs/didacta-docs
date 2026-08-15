# Installation and deployment

Didacta Community is installed on your own infrastructure from the official Docker image [`ghcr.io/va360labs/didacta-community`](https://github.com/va360labs/didacta-io/pkgs/container/didacta-community) (public, no `docker login` required).

## Installation paths

| Path | Who it is for | Guide |
| --- | --- | --- |
| **Docker Compose** (recommended) | Most installations: brings up the app + Postgres + Redis + a test mailbox in one go. | [Install with Docker Compose](docker-compose.md) |
| **Manual docker run** | Operators who already have managed Postgres 16 and Redis 7 and only want the application container. | [Install with docker run](docker-run.md) |

## Requirements

- **Docker 24+** with Docker Compose v2 (`docker compose version`).
- **2 GB of free RAM** and ~2 GB of disk for images and data.
- Free ports (all configurable via environment variables): `3000` (web), `4000` (API), `5432` (Postgres), `6379` (Redis), `8025` (test mailbox UI).

!!! warning "PostgreSQL with pgvector"
    The official compose file uses the `pgvector/pgvector:pg16` image. If you bring your own Postgres it must be **16+ with the `pgvector` extension available** (the AI tutor stores embeddings in `vector` columns). Without it the app will not start (`type "vector" does not exist`).

## Only 3 required variables

Every other variable either has a sensible default or is injected by the compose file. The full reference is in [Environment variables](../configuracion/variables-de-entorno.md).

| Variable | What it is |
| --- | --- |
| `ADMIN_DATABASE_URL` | Administration (superuser) connection string for Postgres 16 with `pgvector`. The app derives its own runtime connection (the `didacta_app` role, without `BYPASSRLS`). |
| `REDIS_URL` | Redis 7 connection string. |
| `AUTH_SECRET` | Secret used to sign sessions and cookies. **At least 32 random characters.** |

!!! tip "Generating `AUTH_SECRET`"
    Any random string of 32+ characters will do:
    ```bash
    openssl rand -base64 32
    # or
    node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"
    ```
    Keep it safe: if you change it, every active session is invalidated.

## Image versioning

Didacta follows **SemVer** and publishes every release as an **immutable** Docker image tag. The `alpha` and `beta` tags track the newest version of their channel, and `latest` will exist only for stable releases. In production, pin your version with `DIDACTA_IMAGE_TAG` in `.env` and decide yourself when to move up — the full policy lives in [Upgrading](actualizacion.md). Published tags are on [GitHub Container Registry](https://github.com/va360labs/didacta-io/pkgs/container/didacta-community).

## After installing

1. [Setup wizard](setup-wizard.md) — create the organization and the first administrator.
2. [Visual walkthrough: getting started](recorrido-visual.md) — screenshot by screenshot, from the wizard to the first certificate issued.
3. [Configuration](../configuracion/index.md) — real SMTP, S3 storage, branding, AI.
4. [Backups](copias-de-seguridad.md) — before there is any real data.
5. [Upgrading](actualizacion.md) — when the next release ships.
