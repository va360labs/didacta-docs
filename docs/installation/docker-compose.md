---
render_macros: true
---

# Instalar con Docker Compose

El camino recomendado. Al terminar tendrás la plataforma completa — web + API + Postgres + Redis + buzón de correo de pruebas — corriendo en tu máquina o servidor.

El stack lo define `docker-compose.alpha.yml`, la experiencia self-host canónica del repositorio.

## Instalar con un comando

Si solo quieres verlo funcionando, el instalador hace los cinco pasos de abajo por ti (necesita Docker con Compose v2 y `curl` o `wget`):

```bash
curl -fsSL https://raw.githubusercontent.com/va360labs/didacta-io/main/install.sh | bash
```

Crea la carpeta `./didacta` (cámbiala con `DIDACTA_DIR`), descarga el compose, **genera `AUTH_SECRET`** y lo guarda en el `.env` (si ya hay uno, no lo toca), fija `DIDACTA_IMAGE_TAG` a la versión del instalador, arranca el stack, espera a que `/healthz` responda y termina imprimiendo el **enlace del asistente** (`http://localhost:3000/setup?token=…`, válido una sola vez), así que no hace falta ir a buscar el token a los logs. Si lo prefieres, léelo antes de ejecutarlo: `curl -fsSL <url> -o install.sh && less install.sh && bash install.sh`.

Variables opcionales del instalador: `DIDACTA_DIR`, `DIDACTA_IMAGE_TAG`, `DIDACTA_COMPOSE_FILE` (un compose local), `WEB_PORT`, `API_PORT`, `MAILPIT_UI_PORT`, y — para **convivir con otra instalación en la misma máquina** — `DIDACTA_PROJECT` (nombre del proyecto de compose), `DIDACTA_COMPOSE_EXTRA` (un fichero de override) y `DIDACTA_NAME_SUFFIX` (sufijo de los nombres de contenedor, por ejemplo `-2`: los `container_name` son globales del host y el nombre de proyecto no los aísla).

## Instalación a mano

```bash
# 1. Descarga el compose y la plantilla de entorno
git clone https://github.com/va360labs/didacta-io.git
cd didacta-io

# 2. Configura el entorno. Con compose solo AUTH_SECRET es obligatoria.
cp .env.example .env
# Edita .env y rellena AUTH_SECRET con un secreto largo y aleatorio:
#   openssl rand -base64 32

# 3. Fija la versión de la imagen (tags publicados en GitHub Container Registry)
echo "DIDACTA_IMAGE_TAG={{ didacta_version }}" >> .env

# 4. Arranca el stack
docker compose -f docker-compose.alpha.yml up -d

# 5. Comprueba que todo está sano (~60-90 s la primera vez)
docker compose -f docker-compose.alpha.yml ps
```

En el primer arranque el contenedor de la app aplica automáticamente las **migraciones versionadas** (`prisma migrate deploy`), las **políticas RLS** y el **seed idempotente de sistema**. No hay que ejecutar nada a mano.

## URLs de la instalación

| URL | Qué es |
| --- | --- |
| `http://localhost:3000` | Web (la primera vez redirige al [asistente de configuración](setup-wizard.md)) |
| `http://localhost:4000/api/docs` | Swagger de la API |
| `http://localhost:4000/healthz` | Probe de salud |
| `http://localhost:8025` | Mailpit — buzón donde caen los emails de prueba |

## Primer acceso

1. Abre el enlace del asistente: `http://localhost:3000/setup?token=…`. El token sale en los logs del contenedor (`docker compose -f docker-compose.alpha.yml logs didacta | grep "Setup token"`) o lo imprime `install.sh`; sin él, el asistente no deja terminar el alta. Ahí creas la organización (tenant) y la cuenta del primer administrador.
2. Entra con esa cuenta y configura tu marca en **Administración → Branding** (logo, colores, textos de la pantalla de acceso).
3. El correo saliente apunta por defecto al buzón de pruebas Mailpit (`http://localhost:8025`). Para producción configura tu SMTP real en **Administración → Configuración → Notificaciones**.

## Persistencia: volúmenes

El compose declara cuatro volúmenes nombrados que sobreviven a `down`/`up` y reinicios:

| Volumen | Qué guarda |
| --- | --- |
| `postgres_data` | Toda la base de datos. |
| `redis_data` | Cola persistente (`appendonly yes`) — outbox + jobs. |
| `didacta_data` (`/app/data`) | Storage local de la aplicación (`/app/data/storage`: uploads de cursos, certificados y evidencias), **la clave de cifrado autogenerada** para datos at-rest (`/app/data/secrets`, **fuera** del árbol que se sirve por HTTP) y la contraseña autogenerada del rol `didacta_app`. |
| `minio_data` | Solo con el profile `s3`. Buckets de MinIO. |

!!! danger "`down -v` borra los datos"
    `docker compose down -v` elimina los volúmenes y, por tanto, la base de datos y los ficheros. Para detener sin borrar usa `docker compose down` **sin** `-v`.

## Pasar más variables al contenedor

El servicio `didacta` del compose **solo recibe las variables que declara en su bloque `environment`** (no lleva `env_file`): `NODE_ENV`, `ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`, `TENANT_SETTINGS_ENC_KEY`, `AUDIT_REPORT_HMAC_KEY`, `METRICS_TOKEN`, `STORAGE_*`, `SMTP_FROM`, `DIDACTA_LICENSE_KEY` y `DIDACTA_DEV_BYPASS`, entre otras. Una variable que escribas en `.env` y no esté ahí **no llega a la aplicación** (de ahí que el SMTP apunte siempre a Mailpit, y que haya que descomentar las `S3_*`). Para añadir otras (`WEB_PUBLIC_URL`, `SMTP_HOST`/`SMTP_PORT`, `STRIPE_*`, `SUPPORT_OPERATOR_ALERT_EMAIL`…), crea un fichero de override y pásalo con `-f`:

```yaml
# docker-compose.override.yml
services:
  didacta:
    environment:
      WEB_PUBLIC_URL: https://campus.ejemplo.com
      SMTP_HOST: smtp.tu-proveedor.com
      SMTP_PORT: '587'
```

```bash
docker compose -f docker-compose.alpha.yml -f docker-compose.override.yml up -d
```

(Con `-f` explícito, compose **no** carga `docker-compose.override.yml` por su cuenta.) El SMTP de la plataforma también puede configurarse por academia desde el panel, sin tocar el compose: ver [Email](../configuration/email.md).

!!! warning "Producción: dos variables que conviene definir"
    Como el compose arranca con `NODE_ENV=production`: **`/metrics` responde 401 hasta que definas `METRICS_TOKEN`** (solo hace falta si lo scrapeas) y **el export firmado del log de auditoría responde 503 hasta que definas `AUDIT_REPORT_HMAC_KEY`** (solo con esa capability Enterprise). Genera ambas con `openssl rand -hex 32`. Y pon `WEB_PUBLIC_URL`: sin ella los correos llevan enlaces a `localhost`.

## Almacenamiento de ficheros

Por defecto los ficheros (portadas, certificados, evidencias) se guardan en disco, en el volumen `didacta_data`. Para usar un almacenamiento S3-compatible:

=== "MinIO local (pruebas)"

    ```bash
    docker compose -f docker-compose.alpha.yml --profile s3 up -d
    # Consola de MinIO en http://localhost:9001
    ```

    Después descomenta las variables `S3_*` del bloque `environment` del servicio `didacta` en el compose.

=== "S3 externo (producción)"

    Rellena las variables `S3_*` en `.env` y pon `STORAGE_DRIVER=s3`. Funciona con AWS S3, Hetzner Object Storage o cualquier proveedor S3-compatible. Detalles en [Almacenamiento](../configuration/storage.md).

## Telemetría

La instalación envía un latido diario anónimo (id de instancia aleatorio + versión + edición + SO) para contar instalaciones vivas. Sin PII y sin efecto si no hay red. Se desactiva con:

```bash
DIDACTA_TELEMETRY_DISABLED=true
```

El detalle del payload está en [Telemetría](../configuration/telemetry.md).

## Licencia Enterprise

Community funciona completa sin licencia. Si tienes una licencia Enterprise, ponla en `DIDACTA_LICENSE_KEY` y las capabilities EE se desbloquean al arrancar. Sin licencia, esas pantallas quedan visibles con un aviso — nunca ocultas. Ver [Enterprise](../enterprise/index.md).

## Siguiente paso

→ [Asistente de configuración](setup-wizard.md), o si algo no arranca, [Solución de problemas](troubleshooting.md).
