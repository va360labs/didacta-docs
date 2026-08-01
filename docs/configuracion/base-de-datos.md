# Base de datos

Didacta usa **PostgreSQL 16** con Prisma y una disciplina estricta de producto: migraciones versionadas, Row-Level Security en toda tabla con `tenant_id` y la extensión `pgvector` para los embeddings de IA.

## Requisitos

- PostgreSQL **16+**.
- Extensión **pgvector** disponible (`CREATE EXTENSION vector;`). El compose oficial usa la imagen `pgvector/pgvector:pg16`, que la trae de serie. También se usan `uuid-ossp` y `pgcrypto`.

## Qué pasa en cada arranque

El entrypoint del contenedor ejecuta, en orden:

1. `CREATE EXTENSION IF NOT EXISTS vector` (aviso no fatal si no puede).
2. **`prisma migrate deploy`** — aplica solo las migraciones pendientes. Si una falla, el arranque se detiene con `P3009` sin tocar datos.
3. **`rls.sql`** — reaplica las políticas de Row-Level Security.
4. **`grants.sql`** — roles y permisos de runtime (crea el rol `didacta_app` sin `BYPASSRLS`; con `POSTGRES_APP_PASSWORD` definida, le asigna contraseña).
5. **`seed.sql`** — seed idempotente de espacios de sistema (no crea datos de demo).

## Row-Level Security autodescubierta

La política RLS **no se declara tabla a tabla**: el script `rls.sql` recorre `information_schema` y aplica a **toda tabla de `public` con columna `tenant_id`** una política `tenant_isolation`:

```sql
CREATE POLICY tenant_isolation ON <tabla>
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());
```

`current_tenant_id()` lee la variable de sesión `app.current_tenant_id`, que la API fija en cada petición según el tenant del token. Cualquier tabla nueva con `tenant_id` recibe RLS automáticamente al reaplicar el script.

### Niveles de enforcement

La variable `RLS_ENFORCEMENT` controla el modo (`off` | `warn` | `on`, por defecto `warn`). Para que el aislamiento sea **real** a nivel de base de datos, `DATABASE_URL` debe conectar con el rol **`didacta_app`** (sin `BYPASSRLS`), no con el superusuario del contenedor.

## Migraciones

- **Todo cambio de schema** viaja en una migración Prisma versionada (`packages/database/prisma/migrations/`). Un self-hoster actualiza entre versiones sin intervención manual.
- `prisma db push` está reservado a desarrollo local; en producción **nunca** se usa.
- Los cambios destructivos (uniques nuevas, drops) llevan siempre plan de migración explícito en el CHANGELOG.

Ver [Actualizar Didacta](../instalacion/actualizacion.md) para el flujo de upgrade y el caso especial del baseline.

## Conexión

```
DATABASE_URL=postgresql://usuario:contraseña@host:5432/didacta?schema=public
```

Con `docker-compose.alpha.yml` se construye automáticamente desde `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB`. El compose publica Postgres **solo en `127.0.0.1`**; si lo expones, cambia la contraseña por defecto.
