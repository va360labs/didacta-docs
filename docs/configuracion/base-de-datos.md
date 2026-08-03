# Base de datos

Didacta usa **PostgreSQL 16** con Prisma y una disciplina estricta de producto: migraciones versionadas, Row-Level Security en toda tabla con `tenant_id` y la extensión `pgvector` para los embeddings de IA.

## Requisitos

- PostgreSQL **16+**.
- Extensión **pgvector** disponible (`CREATE EXTENSION vector;`). El compose oficial usa la imagen `pgvector/pgvector:pg16`, que la trae de serie. También se usan `uuid-ossp` y `pgcrypto`.

## Qué pasa en cada arranque

El entrypoint del contenedor ejecuta, en orden, **con la conexión de administración** (`ADMIN_DATABASE_URL`, ver más abajo):

1. `CREATE EXTENSION IF NOT EXISTS vector` (aviso no fatal si no puede).
2. **`prisma migrate deploy`** — aplica solo las migraciones pendientes. Si una falla, el arranque se detiene con `P3009` sin tocar datos.
3. **`rls.sql`** — reaplica las políticas de Row-Level Security y crea el rol `didacta_super` (`BYPASSRLS`, sin `LOGIN`).
4. **`grants.sql`** — crea el rol de runtime `didacta_app` (sin `BYPASSRLS`) y lo hace miembro de `didacta_super`; le asigna contraseña (`POSTGRES_APP_PASSWORD` si la definiste, o una autogenerada y persistida en el volumen de datos).
5. **`seed.sql`** — seed idempotente de espacios de sistema (no crea datos de demo).

Solo entonces arranca la API, ya conectada con la **conexión de runtime** (`DATABASE_URL`, derivada automáticamente hacia `didacta_app`).

## Row-Level Security autodescubierta

La política RLS **no se declara tabla a tabla**: el script `rls.sql` recorre `information_schema` y aplica a **toda tabla de `public` con columna `tenant_id`** una política `tenant_isolation`:

```sql
CREATE POLICY tenant_isolation ON <tabla>
  USING (tenant_id = current_tenant_id())
  WITH CHECK (tenant_id = current_tenant_id());
```

`current_tenant_id()` lee la variable de sesión `app.current_tenant_id`, que la API fija en cada petición según el tenant del token. Cualquier tabla nueva con `tenant_id` recibe RLS automáticamente al reaplicar el script.

### Niveles de enforcement

La variable `RLS_ENFORCEMENT` controla el modo (`off` | `warn` | `on`, por defecto `on`). En `warn` las queries sin contexto de tenant se loguean a nivel *warning*; en `on`, a nivel *error*. El aislamiento es **real** a nivel de base de datos porque la app conecta en runtime con el rol **`didacta_app`** (sin `BYPASSRLS`), nunca con el superusuario del contenedor — ver § Conexión.

Un puñado de operaciones legítimamente cross-tenant (autenticación por API key, refresh token, resolución de tenant por dominio, el despachador del outbox, el setup wizard) necesitan ver filas de más de un tenant sin conocer ninguno de antemano. En vez de una conexión bypass separada, hacen `SET LOCAL ROLE didacta_super` dentro de su propia transacción — transaccional, sin fugas entre requests, auditado por el código (`runSanctionedGlobalAccess()`).

## Migraciones

- **Todo cambio de schema** viaja en una migración Prisma versionada (`packages/database/prisma/migrations/`). Un self-hoster actualiza entre versiones sin intervención manual.
- `prisma db push` está reservado a desarrollo local; en producción **nunca** se usa.
- Los cambios destructivos (uniques nuevas, drops) llevan siempre plan de migración explícito en el CHANGELOG.

Ver [Actualizar Didacta](../instalacion/actualizacion.md) para el flujo de upgrade y el caso especial del baseline.

## Conexión: administración vs runtime

Dos variables, dos roles distintos:

- **`ADMIN_DATABASE_URL`** — el usuario superuser/owner del cluster. Solo lo usa el entrypoint para migraciones + RLS + grants. Con `docker-compose.alpha.yml` se construye automáticamente desde `POSTGRES_USER`, `POSTGRES_PASSWORD` y `POSTGRES_DB`.
- **`DATABASE_URL`** — la conexión de runtime de la app. **Déjala vacía**: el entrypoint la deriva sola, sustituyendo usuario/contraseña de `ADMIN_DATABASE_URL` por el rol `didacta_app`. La contraseña de ese rol es `POSTGRES_APP_PASSWORD` si la fijas, o se autogenera y persiste en el volumen de datos la primera vez.

```
ADMIN_DATABASE_URL=postgresql://usuario:contraseña@host:5432/didacta?schema=public
DATABASE_URL=
```

El compose publica Postgres **solo en `127.0.0.1`**; si lo expones, cambia la contraseña por defecto. Si tenías una instalación anterior con solo `DATABASE_URL` apuntando al superuser, sigue arrancando sin tocar nada (el entrypoint lo detecta y lo advierte en el log como degradación — sin aislamiento RLS real) — ver [Actualizar Didacta](../instalacion/actualizacion.md).
