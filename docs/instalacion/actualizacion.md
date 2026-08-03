# Actualizar Didacta

Didacta versiona con SemVer y publica cada versión como un tag de imagen Docker (`didactaio/community:<versión>`). **No existe tag `latest`**: cada instalación fija su versión con `DIDACTA_IMAGE_TAG` y decide cuándo subir.

## Actualización normal

```bash
# 1. SIEMPRE: copia de seguridad (ver Copias de seguridad)
docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > pre-upgrade.dump

# 2. Cambia la versión en .env
#    DIDACTA_IMAGE_TAG=<nueva versión>

# 3. Baja la imagen nueva y recrea el contenedor de la app
docker compose -f docker-compose.alpha.yml pull didacta
docker compose -f docker-compose.alpha.yml up -d didacta
```

En el arranque, el contenedor aplica solo las migraciones **pendientes** (`prisma migrate deploy`) y reaplica las políticas RLS. Si una migración falla, el arranque se detiene ruidosamente (error `P3009`) **sin tocar datos**: revisa el log del contenedor, corrige la causa y vuelve a arrancar.

!!! note "Disciplina de schema"
    - El schema nunca se modifica fuera de migraciones versionadas. `prisma db push` está reservado a entornos de desarrollo.
    - No saltes versiones major sin leer el [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md): los breaking changes llevan siempre nota de migración.

## Rollback

1. Restaura la copia:
   ```bash
   docker exec -i didacta-postgres pg_restore -U didacta -d didacta --clean < pre-upgrade.dump
   ```
2. Vuelve a poner el `DIDACTA_IMAGE_TAG` anterior y `docker compose -f docker-compose.alpha.yml up -d didacta`.

!!! danger "Nunca solo la imagen"
    No hagas rollback de la imagen sin restaurar la base de datos: una BD con migraciones de una versión más nueva no es compatible con la imagen antigua.

## Flip a `didacta_app` (aislamiento RLS real)

Desde la versión que introduce el flip, la app deja de conectar con el usuario
bootstrap/superuser en runtime. Si venías de una versión anterior con solo
`DATABASE_URL`, migra tu `.env`:

```bash
# 1. Renombra la línea DATABASE_URL=<valor> a ADMIN_DATABASE_URL=<el mismo valor>
# 2. Borra (o deja vacía) la línea DATABASE_URL

docker compose -f docker-compose.alpha.yml up -d didacta
```

El entrypoint deriva sola la conexión de runtime (rol `didacta_app`, sin
`BYPASSRLS`) a partir de `ADMIN_DATABASE_URL`. El log confirma el modo:
`runtime conecta como didacta_app (aislamiento RLS real)`. Detalle completo en
[Base de datos](../configuracion/base-de-datos.md).

!!! note "No rompe si no migras el .env"
    Si dejas `DATABASE_URL` apuntando al superuser (como antes), el contenedor
    sigue arrancando sin tocar nada — solo pierdes el aislamiento RLS real, y
    el log lo advierte como degradación explícita.

## Caso especial: instalaciones anteriores al baseline (era `db push`)

Hasta la retomada fair-code (2026-07-31) el schema se aplicaba con `prisma db push` y la base de datos **no tiene tabla `_prisma_migrations`**. Esas instalaciones deben adoptar el baseline **una sola vez** antes de arrancar la primera imagen con migraciones:

```bash
# Con la BD de la instalación accesible en DATABASE_URL y la imagen nueva:
docker compose -f docker-compose.alpha.yml run --rm didacta shell
# dentro del contenedor:
pnpm --filter @didacta/database exec prisma migrate resolve --applied 20260731120000_baseline_faircode
exit
docker compose -f docker-compose.alpha.yml up -d didacta
```

`migrate resolve --applied` no ejecuta SQL: solo registra que el schema del baseline ya existe (lo creó `db push` en su día). A partir de ahí, las actualizaciones siguen el camino normal.
