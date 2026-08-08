# Instalar con Docker Compose

El camino recomendado. Al terminar tendrás la plataforma completa — web + API + Postgres + Redis + buzón de correo de pruebas — corriendo en tu máquina o servidor.

El stack lo define `docker-compose.alpha.yml`, la experiencia self-host canónica del repositorio.

## Instalación

```bash
# 1. Descarga el compose y la plantilla de entorno
git clone https://github.com/va360labs/didacta-io.git
cd didacta-io

# 2. Configura el entorno. Con compose solo AUTH_SECRET es obligatoria.
cp .env.example .env
# Edita .env y rellena AUTH_SECRET con un secreto largo y aleatorio:
#   openssl rand -base64 32

# 3. Fija la versión de la imagen (tags publicados en Docker Hub)
echo "DIDACTA_IMAGE_TAG=<versión>" >> .env

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

1. Abre `http://localhost:3000`. La primera vez te llevará al **asistente de configuración** (`/setup`): ahí creas la organización (tenant) y la cuenta del primer administrador.
2. Entra con esa cuenta y configura tu marca en **Administración → Branding** (logo, colores, textos de la pantalla de acceso).
3. El correo saliente apunta por defecto al buzón de pruebas Mailpit (`http://localhost:8025`). Para producción configura tu SMTP real en **Administración → Configuración → Notificaciones**.

## Persistencia: volúmenes

El compose declara cuatro volúmenes nombrados que sobreviven a `down`/`up` y reinicios:

| Volumen | Qué guarda |
| --- | --- |
| `postgres_data` | Toda la base de datos. |
| `redis_data` | Cola persistente (`appendonly yes`) — outbox + jobs. |
| `didacta_data` | Storage local de la aplicación: uploads de cursos, certificados y evidencias **más la clave de cifrado autogenerada** para datos at-rest. |
| `minio_data` | Solo con el profile `s3`. Buckets de MinIO. |

!!! danger "`down -v` borra los datos"
    `docker compose down -v` elimina los volúmenes y, por tanto, la base de datos y los ficheros. Para detener sin borrar usa `docker compose down` **sin** `-v`.

## Almacenamiento de ficheros

Por defecto los ficheros (portadas, certificados, evidencias) se guardan en disco, en el volumen `didacta_data`. Para usar un almacenamiento S3-compatible:

=== "MinIO local (pruebas)"

    ```bash
    docker compose -f docker-compose.alpha.yml --profile s3 up -d
    # Consola de MinIO en http://localhost:9001
    ```

    Después descomenta las variables `S3_*` del bloque `environment` del servicio `didacta` en el compose.

=== "S3 externo (producción)"

    Rellena las variables `S3_*` en `.env` y pon `STORAGE_DRIVER=s3`. Funciona con AWS S3, Hetzner Object Storage o cualquier proveedor S3-compatible. Detalles en [Almacenamiento](../configuracion/almacenamiento.md).

## Telemetría

La instalación envía un latido diario anónimo (id de instancia aleatorio + versión + edición + SO) para contar instalaciones vivas. Sin PII y sin efecto si no hay red. Se desactiva con:

```bash
DIDACTA_TELEMETRY_DISABLED=true
```

El detalle del payload está en [Telemetría](../configuracion/telemetria.md).

## Licencia Enterprise

Community funciona completa sin licencia. Si tienes una licencia Enterprise, ponla en `DIDACTA_LICENSE_KEY` y las capabilities EE se desbloquean al arrancar. Sin licencia, esas pantallas quedan visibles con un aviso — nunca ocultas. Ver [Enterprise](../enterprise/index.md).

## Siguiente paso

→ [Asistente de configuración](setup-wizard.md), o si algo no arranca, [Solución de problemas](solucion-de-problemas.md).
