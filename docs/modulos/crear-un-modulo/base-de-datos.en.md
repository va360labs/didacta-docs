# The module's database

A first-party module's tables live in the host's Prisma schema (`packages/database/prisma/schema.prisma`), not in the module. The module declares its territory through the manifest's `tablePrefix`.

## Mandatory conventions

- The **prefix** `mod_<slug>_` on every table, with `_` instead of `-` (`mod.access-groups` → `mod_access_groups_`).
- **`tenant_id`** on every table. No exceptions for modules.
- **Zero foreign keys** pointing at another module's tables, or at core tables from another domain.
- Schema changes always go in a **versioned migration** (`prisma migrate dev` generates them; `migrate deploy` applies them in production).

## Declaring a table

```prisma
model AccessGroup {
  id        String   @id @default(uuid()) @db.Uuid
  tenantId  String   @map("tenant_id") @db.Uuid
  name      String
  createdAt DateTime @default(now()) @map("created_at")

  @@index([tenantId])
  @@map("mod_access_groups_group")
}
```

`@@map` is what materialises the prefix convention: the Prisma model name is up to you, the table name is not.

## RLS for free

You do not have to write Row-Level Security policies: the host's `rls.sql` script **auto-discovers** every table in `public` with a `tenant_id` column and applies the `tenant_isolation` policy to it (see [Database](../../configuracion/base-de-datos.md)). Your only obligation is that the table carries `tenant_id`; after the migration, run `pnpm db:rls` (in production the entrypoint does it on every start).

## Development workflow

```bash
# 1. Edit packages/database/prisma/schema.prisma

# 2. Generate the versioned migration
pnpm db:migrate          # prisma migrate dev — it will ask you for a name

# 3. Regenerate the client
pnpm db:generate

# 4. Reapply RLS (new tables pick it up here)
pnpm db:rls
```

Name the migration with the module's prefix (`mod_access_groups_membership_source`, `mod_member_registration_datos`…) — that is the convention in the repository's real history.

## Reading another module's data (route B of ADR-016)

A first-party module **may read** another module's tables if — and only if — it:

1. Declares the dependency in the manifest (`dependencies.modules` or `optionalModules`).
2. **Always** filters by `tenant_id`.
3. **Never writes** to those tables and never creates foreign keys to them.

The preferable alternative, when it exists (route A), is to consume the other module's **public service**, injected by the host: for example, `mod.messaging` gets the spaces from `mod.community` by calling `community.listSpaces(tenantId)` instead of reading its tables.

## Data on disable / uninstall

- `onDisable` **preserves** the data: disabling a module is reversible.
- `onUninstall` **archives**, it does not delete: the product's discipline is never to destroy tenant data on its own.
