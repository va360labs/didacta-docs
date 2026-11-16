# Solución de problemas

Problemas frecuentes en la instalación y su arreglo.

| Síntoma | Causa y arreglo |
| --- | --- |
| La app no arranca y el log dice `type "vector" does not exist` | Tu Postgres no tiene pgvector. Usa la imagen `pgvector/pgvector:pg16` o instala la extensión (`CREATE EXTENSION vector;`). |
| `Error: P3009 — migrate found failed migrations` | Una migración quedó a medias (p. ej. por un corte). Revisa el log, corrige la causa y marca la migración con `prisma migrate resolve`; el arranque nunca borra datos por su cuenta. |
| El navegador carga pero la API da 502/timeout | El primer arranque tarda hasta ~60-90 s (healthcheck con `start_period`). Mira `docker compose -f docker-compose.alpha.yml logs -f didacta`. |
| No llegan los correos | En la instalación por defecto los correos van al Mailpit local (`http://localhost:8025`), no a internet. Configura SMTP real en **Administración → Configuración → Notificaciones**. |
| El asistente `/setup` dice que falta el token o es inválido | El alta inicial exige `/setup?token=…`. Mira los logs: `docker compose -f docker-compose.alpha.yml logs didacta \| grep "Setup token"` (si la instancia se reinicia antes de terminar, imprime uno nuevo), o fija `DIDACTA_SETUP_TOKEN` (16+ caracteres). |
| `/metrics` responde 401 tras actualizar | En producción `/metrics` exige `METRICS_TOKEN`. Defínelo y pásaselo a Prometheus como `Authorization: Bearer …`. |
| El export firmado del audit log responde 503 | Falta `AUDIT_REPORT_HMAC_KEY` (obligatoria con `NODE_ENV=production`). |
| La API no arranca y habla de la clave de cifrado | En producción, si no hay `TENANT_SETTINGS_ENC_KEY` y el fichero de la clave no se puede leer ni crear, **no arranca**. Comprueba que `/app/data` es un volumen **escribible** o fija la clave tú (`openssl rand -hex 32`). Si el volumen está montado exactamente en `STORAGE_ROOT`, define `SECRETS_DIR` en otro volumen. |
| Una dirección o un subdominio responde 404 | Didacta responde 404 a un host que no pertenece a ninguna academia. Da de alta el dominio (**Administración → Tenants**) o, si necesitas servir cualquier host, `DIDACTA_ALLOW_UNKNOWN_HOSTS=true`. Ver [Branding → Dominio propio](../configuration/branding.md#dominio-propio). |
| Un correo llega con enlaces a `localhost` | Falta `WEB_PUBLIC_URL` y la academia no tiene dominio primario verificado. Con compose, hay que pasarla al contenedor (ver [Pasar más variables](docker-compose.md#pasar-mas-variables-al-contenedor)). |
| Una variable del `.env` no tiene efecto con compose | El servicio `didacta` solo recibe las variables que su bloque `environment` declara; ver [Pasar más variables](docker-compose.md#pasar-mas-variables-al-contenedor). |
| Muchos `429` | Límites por minuto de Community (100 por academia con sesión, 30 por IP sin sesión) y límite propio del acceso (por IP y por cuenta). Ver `RATE_LIMIT_*` en [Variables de entorno](../configuration/environment-variables.md). Con un proxy mal configurado (`TRUSTED_PROXY_HOPS`) todos los visitantes comparten IP. |
| Puerto ocupado | Cambia `WEB_PORT`, `API_PORT`, `POSTGRES_PORT`… en `.env`. |
| La app arranca pero no puedo iniciar sesión / la sesión se cae | Comprueba que `AUTH_SECRET` tiene **32+ caracteres** y no ha cambiado entre reinicios (cambiarlo invalida todas las sesiones). |

## Dónde mirar

```bash
# Estado de los servicios y healthchecks
docker compose -f docker-compose.alpha.yml ps

# Logs de la aplicación (bootstrap, migraciones, errores)
docker compose -f docker-compose.alpha.yml logs -f didacta

# Salud de la API
curl -fsS http://localhost:4000/healthz
```

## Pedir ayuda

- **Bugs y feedback**: issues en [GitHub](https://github.com/va360labs/didacta-io/issues) — hay plantillas de bug, feedback y feature request. Antes de abrir una, mira [Reportar un error](../community/reporting-a-bug.md): qué datos hacen falta y qué anonimizar.
- **Vulnerabilidades de seguridad**: sigue la política de [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md) — nunca las publiques en un issue.
