# Base de datos del módulo

Las tablas de un módulo first-party viven en el schema Prisma del host (`packages/database/prisma/schema.prisma`), no en el módulo. El módulo declara su territorio con el `tablePrefix` del manifest.

## Convenciones obligatorias

- **Prefijo** `mod_<slug>_` en toda tabla, con `_` en lugar de `-` (`mod.access-groups` → `mod_access_groups_`).
- **`tenant_id`** en toda tabla. Sin excepciones para módulos.
- **Cero foreign keys** hacia tablas de otro módulo o del core de otro dominio.
- Cambios de schema siempre en **migración versionada** (`prisma migrate dev` genera; `migrate deploy` aplica en producción).

## Declarar una tabla

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

`@@map` es lo que materializa la convención de prefijo: el nombre del modelo Prisma es libre, el de la tabla no.

## RLS gratis

No tienes que escribir políticas de Row-Level Security: el script `rls.sql` del host **autodescubre** toda tabla de `public` con columna `tenant_id` y le aplica la política `tenant_isolation` (ver [Base de datos](../../configuracion/base-de-datos.md)). Tu única obligación es que la tabla lleve `tenant_id`; tras la migración, ejecuta `pnpm db:rls` (en producción lo hace el entrypoint en cada arranque).

## Flujo de trabajo en desarrollo

```bash
# 1. Edita packages/database/prisma/schema.prisma

# 2. Genera la migración versionada
pnpm db:migrate          # prisma migrate dev — te pedirá nombre

# 3. Regenera el cliente
pnpm db:generate

# 4. Reaplica RLS (las tablas nuevas la reciben aquí)
pnpm db:rls
```

Nombra la migración con el prefijo del módulo (`mod_access_groups_membership_source`, `mod_member_registration_datos`…) — es la convención del historial real del repo.

## Leer datos de otro módulo (vía B del ADR-016)

Un módulo first-party **puede leer** tablas de otro módulo si — y solo si —:

1. Declara la dependencia en el manifest (`dependencies.modules` u `optionalModules`).
2. Filtra **siempre** por `tenant_id`.
3. **Jamás escribe** en esas tablas ni les crea FKs.

La alternativa preferible cuando existe (vía A) es consumir el **service público** del otro módulo, inyectado por el host: por ejemplo, `mod.messaging` obtiene los espacios de `mod.community` llamando a `community.listSpaces(tenantId)` en lugar de leer sus tablas.

## Datos al desactivar / desinstalar

- `onDisable` **conserva** los datos: desactivar un módulo es reversible.
- `onUninstall` **archiva**, no borra: la disciplina del producto es no destruir datos de tenant por su cuenta.
