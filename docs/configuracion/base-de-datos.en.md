# Database

Didacta uses **PostgreSQL 16** with Prisma and strict product discipline: versioned migrations, Row-Level Security on every table with a `tenant_id`, and the `pgvector` extension for AI embeddings.

## Requirements

- PostgreSQL **16+**.
- The **pgvector** extension available (`CREATE EXTENSION vector;`). The official compose file uses the `pgvector/pgvector:pg16` image, which ships with it. `uuid-ossp` and `pgcrypto` are used as well.

## What happens on every start

The container entrypoint runs, in order, **using the administration connection** (`ADMIN_DATABASE_URL`, see below):

1. `CREATE EXTENSION IF NOT EXISTS vector` (a non-fatal warning if it cannot).
2. **`prisma migrate deploy`** — applies only the pending migrations. If one fails, startup stops with `P3009` without touching data.
3. **`rls.sql`** — reapplies the Row-Level Security policies and creates the `didacta_super` role (`BYPASSRLS`, no `LOGIN`).
4. **`grants.sql`** — creates the `didacta_app` runtime role (without `BYPASSRLS`) and makes it a member of `didacta_super`; it assigns its password (`POSTGRES_APP_PASSWORD` if you defined one, or an auto-generated one persisted in the data volume).
5. **`seed.sql`** — an idempotent seed of system spaces (it creates no demo data).

Only then does the API start, already connected through the **runtime connection** (`DATABASE_URL`, derived automatically towards `didacta_app`).

## Auto-discovered Row-Level Security

The RLS policy is **not declared table by table**: the `rls.sql` script walks `information_schema` and applies a `tenant_isolation` policy to **every table in `public` that has a `tenant_id` column**:

```sql
CREATE POLICY tenant_isolation ON <table>
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());
```

`current_tenant_id()` reads the `app.current_tenant_id` session variable, which the API sets on every request from the token's tenant. Any new table with a `tenant_id` gets RLS automatically when the script is reapplied.

### Enforcement levels

The `RLS_ENFORCEMENT` variable controls the mode (`off` | `warn` | `on`, default `on`). In `warn`, queries without a tenant context are logged at *warning* level; in `on`, at *error* level. Isolation is **real** at the database level because at runtime the app connects with the **`didacta_app`** role (without `BYPASSRLS`), never with the container superuser — see § Connection.

A handful of legitimately cross-tenant operations (API key authentication, refresh tokens, resolving a tenant by domain, the outbox dispatcher, the setup wizard) need to see rows from more than one tenant without knowing any of them in advance. Instead of a separate bypass connection, they run `SET LOCAL ROLE didacta_super` inside their own transaction — transactional, with no leakage between requests, and audited by the code (`runSanctionedGlobalAccess()`).

## Migrations

- **Every schema change** travels in a versioned Prisma migration (`packages/database/prisma/migrations/`). A self-hoster upgrades between versions with no manual intervention.
- `prisma db push` is reserved for local development; it is **never** used in production.
- Destructive changes (new unique constraints, drops) always come with an explicit migration plan in the CHANGELOG.

See [Upgrading Didacta](../instalacion/actualizacion.md) for the upgrade flow and the special baseline case.

## Connection: administration vs runtime

Two variables, two different roles:

- **`ADMIN_DATABASE_URL`** — the cluster's superuser/owner account. It is used only by the entrypoint, for migrations + RLS + grants. With `docker-compose.alpha.yml` it is built automatically from `POSTGRES_USER`, `POSTGRES_PASSWORD` and `POSTGRES_DB`.
- **`DATABASE_URL`** — the app's runtime connection. **Leave it empty**: the entrypoint derives it on its own, swapping the username/password from `ADMIN_DATABASE_URL` for the `didacta_app` role. That role's password is `POSTGRES_APP_PASSWORD` if you set one, or it is auto-generated and persisted in the data volume on first start.

```
ADMIN_DATABASE_URL=postgresql://user:password@host:5432/didacta?schema=public
DATABASE_URL=
```

The compose file publishes Postgres **on `127.0.0.1` only**; if you expose it, change the default password. If you had an earlier installation with only `DATABASE_URL` pointing at the superuser, it still starts with no changes (the entrypoint detects it and warns in the log as a degradation — no real RLS isolation) — see [Upgrading Didacta](../instalacion/actualizacion.md).
