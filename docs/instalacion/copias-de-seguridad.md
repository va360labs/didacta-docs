# Copias de seguridad

Lo que hay que salvar son **dos cosas**: la base de datos y el volumen de ficheros (que además contiene la clave de cifrado autogenerada para datos at-rest).

## Base de datos

```bash
docker exec didacta-postgres pg_dump -U didacta -d didacta -Fc > didacta-$(date +%F).dump
```

## Ficheros (storage local + clave de cifrado)

```bash
docker run --rm -v didacta-io_didacta_data:/data -v "$PWD":/backup alpine \
  tar czf /backup/didacta-data-$(date +%F).tar.gz -C /data .
```

!!! tip "Nombre del volumen"
    Docker Compose prefija los volúmenes con el nombre del proyecto (la carpeta del compose). Comprueba el nombre real con `docker volume ls | grep didacta_data` y ajústalo en el comando.

Si usas storage S3, el proveedor ya se ocupa de la durabilidad de los ficheros, pero **la clave de cifrado sigue viviendo en el volumen local**: inclúyelo igualmente en la copia.

## Restaurar

```bash
# Base de datos
docker exec -i didacta-postgres pg_restore -U didacta -d didacta --clean < didacta-YYYY-MM-DD.dump

# Ficheros
docker run --rm -v didacta-io_didacta_data:/data -v "$PWD":/backup alpine \
  tar xzf /backup/didacta-data-YYYY-MM-DD.tar.gz -C /data
```

## Cuándo

- **Siempre antes de actualizar de versión** (ver [Actualizar](actualizacion.md)).
- En producción, programa la copia diaria (cron en el host) y guarda las copias fuera de la máquina.
