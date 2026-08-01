# Almacenamiento de ficheros

Didacta guarda ficheros — portadas de curso, certificados PDF, paquetes SCORM, evidencias Fundae, logos — en un **storage conmutables por driver**: disco local (por defecto) o cualquier proveedor **S3-compatible**.

## Driver local (por defecto)

Los ficheros van al volumen de datos, bajo `STORAGE_ROOT` (`/app/data/storage` en la imagen oficial).

!!! warning "El volumen de datos guarda algo más que ficheros"
    En `STORAGE_ROOT` vive también la **clave de cifrado autogenerada** (`.didacta-secret-key`) que protege los secretos por tenant (SSO, SMTP, Stripe, Zoom…). Aunque uses S3 para los ficheros, **mantén el volumen montado y respaldado**, o fija la clave con `TENANT_SETTINGS_ENC_KEY`.

## Driver S3

Configura las cuatro variables y el driver:

```bash
STORAGE_DRIVER=s3
S3_ENDPOINT=https://s3.eu-central-1.amazonaws.com   # o tu MinIO/Hetzner/otro
S3_BUCKET=didacta
S3_ACCESS_KEY=...
S3_SECRET_KEY=...
# Opcionales:
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true        # necesario para MinIO
S3_PRESIGNED_TTL_SECONDS=...    # TTL de las URLs prefirmadas
```

Comportamiento del selector `STORAGE_DRIVER`:

- `local` — fuerza disco.
- `s3` — fuerza S3 y **falla al arrancar** si falta alguna de las 4 variables (mejor un error ruidoso que subir ficheros a un limbo).
- *(sin definir)* — intenta S3 y cae a disco local si la configuración está incompleta.

## MinIO para pruebas

El compose oficial trae MinIO bajo el profile `s3`:

```bash
docker compose -f docker-compose.alpha.yml --profile s3 up -d
# API S3:    http://localhost:9000
# Consola:   http://localhost:9001  (didacta / didacta_dev)
```

El job `minio-init` crea el bucket `didacta-dev` automáticamente. Después, descomenta el bloque `S3_*` del servicio `didacta` en el compose.

## Copias de seguridad

- **Driver local**: respalda el volumen `didacta_data` completo (ficheros + clave de cifrado).
- **Driver S3**: el proveedor se ocupa de la durabilidad de los ficheros, pero la clave de cifrado sigue en el volumen local — inclúyelo igualmente. Ver [Copias de seguridad](../instalacion/copias-de-seguridad.md).
