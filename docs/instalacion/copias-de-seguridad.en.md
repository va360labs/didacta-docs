# Backups

There are **two things** to save: the database and the file volume (which also contains the auto-generated encryption key for data at rest).

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
    Docker Compose prefixes volumes with the project name (the folder containing the compose file). Check the real name with `docker volume ls | grep didacta_data` and adjust the command accordingly.

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

- **Always before upgrading to a new version** (see [Upgrading](actualizacion.md)).
- In production, schedule a daily backup (cron on the host) and keep the copies off the machine.
