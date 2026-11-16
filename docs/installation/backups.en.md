# Backups

There are **two things** to save: the database and the file volume (`/app/data`, which holds the local storage **and** the auto-generated encryption key for data at rest, at `/app/data/secrets/.didacta-secret-key`).

!!! danger "Without the key nothing can be decrypted"
    A copy of the database **without** that key leaves the encrypted credentials unreadable (Stripe, per-tenant SMTP, SSO, Zoom…). If you set `TENANT_SETTINGS_ENC_KEY` in the environment, keep it in your secrets manager; if not, the volume copy is the only one that has it.

## Database

```bash
docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > didacta-$(date +%F).dump
```

## Files (local storage + encryption key)

```bash
docker run --rm -v didacta-io_didacta_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/didacta-data-$(date +%F).tar.gz -C /data .
```

!!! tip "The volume name"
    Docker Compose prefixes volumes with the project name (the folder containing the compose file). With `install.sh` (folder `didacta`) the volume is `didacta_didacta_data`; with a repository clone, `didacta-io_didacta_data`. Check the real name with `docker volume ls | grep didacta_data` and adjust the command accordingly. The containers are called `didacta-postgres` and `didacta-app` unless you set `DIDACTA_NAME_SUFFIX`.

If you use S3 storage, the provider already takes care of file durability, but **the encryption key still lives in the local volume**: include it in the backup anyway.

## Restoring

```bash
# Database
docker exec -i didacta-postgres pg_restore -U didacta -d didacta --clean < didacta-YYYY-MM-DD.dump

# Files
docker run --rm -v didacta-io_didacta_data:/data -v "$PWD":/backup alpine \
  tar xzf /backup/didacta-data-YYYY-MM-DD.tar.gz -C /data
```

## When

- **Always before upgrading to a new version** (see [Upgrading](upgrading.md)).
- In production, schedule a daily backup (cron on the host) and keep the copies off the machine.
