# Copias de seguridad

Lo que hay que salvar son **dos cosas**: la base de datos y el volumen de ficheros (`/app/data`, que contiene el storage local **y** la clave de cifrado autogenerada para datos at-rest, en `/app/data/secrets/.didacta-secret-key`).

!!! danger "Sin la clave no se descifra nada"
    Una copia de la base de datos **sin** esa clave deja ilegibles las credenciales cifradas (Stripe, SMTP por tenant, SSO, Zoom…). Si fijaste `TENANT_SETTINGS_ENC_KEY` en el entorno, guárdala en tu gestor de secretos; si no, la copia del volumen es la única que la tiene.

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
    Docker Compose prefija los volúmenes con el nombre del proyecto (la carpeta del compose). Con `install.sh` (carpeta `didacta`) el volumen es `didacta_didacta_data`; con un clon del repo, `didacta-io_didacta_data`. Comprueba el nombre real con `docker volume ls | grep didacta_data` y ajústalo en el comando. Los contenedores se llaman `didacta-postgres` y `didacta-app` salvo que hayas fijado `DIDACTA_NAME_SUFFIX`.

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

- **Siempre antes de actualizar de versión** (ver [Actualizar](upgrading.md)).
- En producción, programa la copia diaria (cron en el host) y guarda las copias fuera de la máquina.
