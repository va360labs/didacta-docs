---
render_macros: true
---

# Actualizar Didacta — política de versiones y actualizaciones

Didacta versiona con **SemVer** (`X.Y.Z[-canal.N]`) y publica cada versión como
un tag **inmutable** de la imagen `ghcr.io/va360labs/didacta-community`. Esta
página es la política completa: qué significa cada canal, qué etiqueta usar en
cada sitio y cómo se sube (y se vuelve) de versión.

## Canales

| Canal | Sufijo | Qué esperar |
| --- | --- | --- |
| **Alpha** | `-alpha.N` | Todo puede cambiar entre versiones, API incluida. Para probar y contarnos fallos. |
| **Beta** | `-beta.N` | El producto completo, en uso real. Puede haber cambios entre versiones; cada release los cuenta en sus notas. |
| **Estable** | sin sufijo | Los cambios incompatibles llegan solo con un salto de versión y con nota de migración. |

Antes de la 1.0, los cambios incompatibles **suben el número menor** (0.1 →
0.2); el sufijo solo marca el canal. Cada versión se publica en
[GitHub Releases](https://github.com/va360labs/didacta-io/releases) con sus
notas — léelas antes de subir.

**Soporte:** mientras no lleguemos a la 1.0 damos soporte solo a la **última
versión publicada**. Si un fallo aparece en una versión anterior, te pediremos
actualizar primero.

## Etiquetas Docker: cuál usar

| Etiqueta | Qué es | Para qué sirve |
| --- | --- | --- |
| `:{{ didacta_version }}` (una versión concreta) | Inmutable: siempre la misma imagen. | **Producción. Usa siempre esta.** |
| `:alpha` / `:beta` | Móvil: sigue a la última versión de su canal. | Entornos de prueba que quieran ir al día. |
| `:latest` | Móvil: solo versiones **estables**. No existe hasta la primera estable. | Producción que acepte subir al ritmo del proyecto, cuando haya estables. |

!!! warning "En producción, versión fijada"
    Un tag móvil significa actualizarse sin leer las notas y sin copia de
    seguridad previa — con migraciones automáticas de base de datos, es la
    receta del disgusto. Fija `DIDACTA_IMAGE_TAG` a una versión concreta y
    sube tú, cuando quieras, con el checklist de abajo.

## Antes de actualizar

1. **Copia de seguridad, siempre** (ver [Copias de seguridad](copias-de-seguridad.md)):
   ```bash
   docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > pre-upgrade.dump
   ```
2. **Lee las notas de la release** en GitHub: los cambios incompatibles llevan
   siempre nota de migración.
3. Comprueba que sabes qué versión corres (`/healthz` la devuelve) por si hay
   que volver.

## Actualización normal

```bash
# 1. Cambia la versión en .env (la última publicada es {{ didacta_version }})
#    DIDACTA_IMAGE_TAG={{ didacta_version }}

# 2. Baja la imagen nueva y recrea el contenedor de la app
docker compose -f docker-compose.alpha.yml pull didacta
docker compose -f docker-compose.alpha.yml up -d didacta
```

En el arranque, el contenedor aplica solo las migraciones **pendientes**
(`prisma migrate deploy`) y reaplica las políticas RLS. Si una migración falla,
el arranque se detiene ruidosamente (error `P3009`) **sin tocar datos**: revisa
el log del contenedor, corrige la causa y vuelve a arrancar.

!!! note "Disciplina de schema"
    - El schema nunca se modifica fuera de migraciones versionadas. `prisma db push` está reservado a entornos de desarrollo.
    - No saltes versiones sin leer las notas de cada release intermedia: los breaking changes llevan siempre nota de migración.

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
