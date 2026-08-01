# Instalar con docker run

Para operadores que ya tienen **Postgres 16 y Redis 7 administrados** (RDS, Cloud SQL, un Postgres propio…) y solo quieren ejecutar el contenedor de la aplicación.

## Requisitos previos

- Postgres 16 con la extensión **pgvector** instalada y un schema vacío. La aplicación aplica las migraciones Prisma al arrancar.
- Redis 7 accesible desde el contenedor.
- Las 3 variables obligatorias: `DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET` (32+ caracteres).

## Ejecución

```bash
docker pull didactaio/community:<versión>

# Volumen para uploads + clave de cifrado autogenerada.
docker volume create didacta_data

docker run -d \
  --name didacta-app \
  -p 3000:3000 \
  -p 4000:4000 \
  -v didacta_data:/app/data \
  -e DATABASE_URL='postgresql://USER:PASS@HOST:5432/didacta?schema=public' \
  -e REDIS_URL='redis://HOST:6379' \
  -e AUTH_SECRET='cualquier-cadena-aleatoria-de-32+-caracteres' \
  -e STORAGE_DRIVER=local \
  -e STORAGE_ROOT=/app/data/storage \
  -e NODE_ENV=production \
  --restart unless-stopped \
  didactaio/community:<versión>
```

!!! warning "El volumen `didacta_data` no es opcional en la práctica"
    Guarda los archivos subidos — cursos, certificados, evidencias — **y** una clave de cifrado autogenerada en el primer arranque para los secretos at-rest. Sin ese volumen montado, todo se pierde al recrear el contenedor.

    Si usas S3 en lugar de disco local (elimina `STORAGE_DRIVER`/`STORAGE_ROOT` y añade `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY`), **mantén igualmente el volumen montado** para la clave de cifrado.

## Verificar

```bash
docker logs -f didacta-app                  # bootstrap + migraciones Prisma
curl -fsS http://localhost:4000/healthz     # debe responder 200
```

Después abre `http://localhost:3000` y sigue el [asistente de configuración](setup-wizard.md).

## Variables opcionales útiles

| Variable | Para qué |
| --- | --- |
| `S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | Storage S3-compatible (MinIO, AWS, Hetzner…). |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | Emails transaccionales. Sin esto, los emails se registran en logs pero no se envían. |
| `DIDACTA_LICENSE_KEY` | JWT firmado para activar capabilities Enterprise. Sin él, modo Community puro. |
| `METRICS_TOKEN` | Bearer token que protege `/metrics` (Prometheus). Vacío = endpoint público. |

La referencia 1 a 1 de todas las variables está en [Variables de entorno](../configuracion/variables-de-entorno.md).
