# Upgrading Didacta

Didacta follows SemVer and publishes every release as a Docker image tag (`didactaio/community:<version>`). **There is no `latest` tag**: each installation pins its version with `DIDACTA_IMAGE_TAG` and decides when to move up.

## Normal upgrade

```bash
# 1. ALWAYS: take a backup first (see Backups)
docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > pre-upgrade.dump

# 2. Change the version in .env
#    DIDACTA_IMAGE_TAG=<new version>

# 3. Pull the new image and recreate the app container
docker compose -f docker-compose.alpha.yml pull didacta
docker compose -f docker-compose.alpha.yml up -d didacta
```

At startup the container applies only the **pending** migrations (`prisma migrate deploy`) and reapplies the RLS policies. If a migration fails, startup stops loudly (error `P3009`) **without touching data**: check the container log, fix the cause and start it again.

!!! note "Schema discipline"
    - The schema is never modified outside versioned migrations. `prisma db push` is reserved for development environments.
    - Do not skip major versions without reading the [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md): breaking changes always ship with a migration note.

## Rollback

1. Restore the backup:
   ```bash
   docker exec -i didacta-postgres pg_restore -U didacta -d didacta --clean < pre-upgrade.dump
   ```
2. Set `DIDACTA_IMAGE_TAG` back to the previous value and run `docker compose -f docker-compose.alpha.yml up -d didacta`.

!!! danger "Never roll back the image alone"
    Do not roll back the image without restoring the database: a database carrying migrations from a newer version is not compatible with the older image.

## Switching to `didacta_app` (real RLS isolation)

As of the release that introduces this switch, the app no longer connects with the
bootstrap/superuser account at runtime. If you are coming from an earlier version with
only `DATABASE_URL`, migrate your `.env`:

```bash
# 1. Rename the DATABASE_URL=<value> line to ADMIN_DATABASE_URL=<the same value>
# 2. Delete (or leave empty) the DATABASE_URL line

docker compose -f docker-compose.alpha.yml up -d didacta
```

The entrypoint derives the runtime connection on its own (the `didacta_app` role, without
`BYPASSRLS`) from `ADMIN_DATABASE_URL`. The log confirms the mode:
`runtime conecta como didacta_app (aislamiento RLS real)`. Full detail in
[Database](../configuracion/base-de-datos.md).

!!! note "Nothing breaks if you do not migrate the .env"
    If you leave `DATABASE_URL` pointing at the superuser (as before), the container
    still starts and nothing changes — you simply lose real RLS isolation, and
    the log warns about it as an explicit degradation.

## Special case: installations older than the baseline (the `db push` era)

Until the fair-code restart (2026-07-31) the schema was applied with `prisma db push` and the database **has no `_prisma_migrations` table**. Those installations must adopt the baseline **once** before starting the first image that carries migrations:

```bash
# With the installation's database reachable at DATABASE_URL and the new image:
docker compose -f docker-compose.alpha.yml run --rm didacta shell
# inside the container:
pnpm --filter @didacta/database exec prisma migrate resolve --applied 20260731120000_baseline_faircode
exit
docker compose -f docker-compose.alpha.yml up -d didacta
```

`migrate resolve --applied` runs no SQL: it simply records that the baseline schema already exists (`db push` created it back in the day). From then on, upgrades follow the normal path.
