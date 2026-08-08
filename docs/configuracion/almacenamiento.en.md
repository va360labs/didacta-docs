# File storage

Didacta stores files — course cover images, PDF certificates, SCORM packages, Fundae evidence, logos — in **driver-switchable storage**: local disk (the default) or any **S3-compatible** provider.

## Local driver (default)

Files go to the data volume, under `STORAGE_ROOT` (`/app/data/storage` in the official image).

!!! warning "The data volume holds more than files"
    `STORAGE_ROOT` is also where the **auto-generated encryption key** (`.didacta-secret-key`) lives, protecting per-tenant secrets (SSO, SMTP, Stripe, Zoom…). Even if you use S3 for files, **keep the volume mounted and backed up**, or pin the key with `TENANT_SETTINGS_ENC_KEY`.

## S3 driver

Configure the four variables and the driver:

```bash
STORAGE_DRIVER=s3
S3_ENDPOINT=https://s3.eu-central-1.amazonaws.com   # or your MinIO/Hetzner/other
S3_BUCKET=didacta
S3_ACCESS_KEY=...
S3_SECRET_KEY=...
# Optional:
S3_REGION=us-east-1
S3_FORCE_PATH_STYLE=true        # required for MinIO
S3_PRESIGNED_TTL_SECONDS=...    # TTL of the presigned URLs
```

How the `STORAGE_DRIVER` selector behaves:

- `local` — forces disk.
- `s3` — forces S3 and **fails at startup** if any of the 4 variables is missing (better a loud error than uploading files into limbo).
- *(unset)* — tries S3 and falls back to local disk if the configuration is incomplete.

## MinIO for testing

The official compose file ships MinIO under the `s3` profile:

```bash
docker compose -f docker-compose.alpha.yml --profile s3 up -d
# S3 API:    http://localhost:9000
# Console:   http://localhost:9001  (didacta / didacta_dev)
```

The `minio-init` job creates the `didacta-dev` bucket automatically. Then uncomment the `S3_*` block of the `didacta` service in the compose file.

## Backups

- **Local driver**: back up the whole `didacta_data` volume (files + encryption key).
- **S3 driver**: the provider takes care of file durability, but the encryption key still lives in the local volume — include it anyway. See [Backups](../instalacion/copias-de-seguridad.md).
