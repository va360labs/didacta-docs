---
render_macros: true
---

# Upgrading Didacta — versioning and update policy

Didacta follows **SemVer** (`X.Y.Z[-channel.N]`) and publishes every release
as an **immutable** tag of the `ghcr.io/va360labs/didacta-community` image.
This page is the full policy: what each channel means, which tag to use where,
and how to move up (and back) a version.

## Channels

| Channel | Suffix | What to expect |
| --- | --- | --- |
| **Alpha** | `-alpha.N` | Anything can change between versions, API included. For testing and telling us about bugs. |
| **Beta** | `-beta.N` | The full product, in real use. There can be changes between versions; every release lists them in its notes. |
| **Stable** | no suffix | Breaking changes only arrive with a version bump, always with a migration note. |

Before 1.0, breaking changes **bump the minor number** (0.1 → 0.2); the suffix
only marks the channel. Every version is published on
[GitHub Releases](https://github.com/va360labs/didacta-io/releases) with its
notes — read them before upgrading.

**Support:** until we reach 1.0 we support only the **latest published
version**. If a flaw shows up in an older version, we will ask you to upgrade
first.

## Docker tags: which one to use

| Tag | What it is | What it is for |
| --- | --- | --- |
| `:{{ didacta_version }}` (a specific version) | Immutable: always the same image. | **Production. Always use this one.** |
| `:alpha` / `:beta` | Moving: tracks the newest version of its channel. | Test environments that want to stay current. |
| `:latest` | Moving: **stable** versions only. Does not exist until the first stable release. | Production that accepts upgrading at the project's pace, once stables exist. |

!!! warning "In production, pin the version"
    A moving tag means upgrading without reading the notes and without a prior
    backup — with automatic database migrations, that is a recipe for a bad
    day. Pin `DIDACTA_IMAGE_TAG` to a specific version and upgrade yourself,
    when you choose, with the checklist below.

## Before you upgrade

1. **Back up, always** (see [Backups](copias-de-seguridad.md)):
   ```bash
   docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > pre-upgrade.dump
   ```
2. **Read the release notes** on GitHub: breaking changes always ship with a
   migration note.
3. Make sure you know which version you are running (`/healthz` returns it) in
   case you need to go back.

## Normal upgrade

```bash
# 1. Change the version in .env (the newest published one is {{ didacta_version }})
#    DIDACTA_IMAGE_TAG={{ didacta_version }}

# 2. Pull the new image and recreate the app container
docker compose -f docker-compose.alpha.yml pull didacta
docker compose -f docker-compose.alpha.yml up -d didacta
```

At startup the container applies only the **pending** migrations (`prisma migrate deploy`) and reapplies the RLS policies. If a migration fails, startup stops loudly (error `P3009`) **without touching data**: check the container log, fix the cause and start it again.

!!! note "Schema discipline"
    - The schema is never modified outside versioned migrations. `prisma db push` is reserved for development environments.
    - Do not skip versions without reading the notes of each release in between: breaking changes always ship with a migration note.

## Rollback

1. Restore the backup:
   ```bash
   docker exec -i didacta-postgres pg_restore -U didacta -d didacta --clean < pre-upgrade.dump
   ```
2. Set `DIDACTA_IMAGE_TAG` back to the previous value and run `docker compose -f docker-compose.alpha.yml up -d didacta`.

!!! danger "Never roll back the image alone"
    Do not roll back the image without restoring the database: a database carrying migrations from a newer version is not compatible with the older image.

## Flip to `didacta_app` (real RLS isolation)

Since the version that introduces the flip, the app no longer connects with the
bootstrap/superuser account at runtime. If you come from an older version with
only `DATABASE_URL`, migrate your `.env`:

```bash
# 1. Rename the DATABASE_URL=<value> line to ADMIN_DATABASE_URL=<same value>
# 2. Delete (or leave empty) the DATABASE_URL line

docker compose -f docker-compose.alpha.yml up -d didacta
```

The entrypoint derives the runtime connection (role `didacta_app`, without
`BYPASSRLS`) from `ADMIN_DATABASE_URL` on its own. The log confirms the mode:
`runtime conecta como didacta_app (aislamiento RLS real)`. Full detail in
[Database](../configuracion/base-de-datos.md).

!!! note "Nothing breaks if you do not migrate the .env"
    If you leave `DATABASE_URL` pointing at the superuser (as before), the
    container still starts without touching anything — you only lose real RLS
    isolation, and the log calls it out as an explicit degradation.

## Special case: installations older than the baseline (the `db push` era)

Until the fair-code relaunch (2026-07-31) the schema was applied with `prisma db push` and the database **has no `_prisma_migrations` table**. Those installations must adopt the baseline **once** before starting the first image with migrations:

```bash
# With the installation's DB reachable at DATABASE_URL and the new image:
docker compose -f docker-compose.alpha.yml run --rm didacta shell
# inside the container:
pnpm --filter @didacta/database exec prisma migrate resolve --applied 20260731120000_baseline_faircode
exit
docker compose -f docker-compose.alpha.yml up -d didacta
```

`migrate resolve --applied` runs no SQL: it only records that the baseline schema already exists (`db push` created it back in the day). From there on, upgrades follow the normal path.
