# Solución de problemas

Problemas frecuentes en la instalación y su arreglo.

| Síntoma | Causa y arreglo |
| --- | --- |
| La app no arranca y el log dice `type "vector" does not exist` | Tu Postgres no tiene pgvector. Usa la imagen `pgvector/pgvector:pg16` o instala la extensión (`CREATE EXTENSION vector;`). |
| `Error: P3009 — migrate found failed migrations` | Una migración quedó a medias (p. ej. por un corte). Revisa el log, corrige la causa y marca la migración con `prisma migrate resolve`; el arranque nunca borra datos por su cuenta. |
| El navegador carga pero la API da 502/timeout | El primer arranque tarda hasta ~60-90 s (healthcheck con `start_period`). Mira `docker compose -f docker-compose.alpha.yml logs -f didacta`. |
| No llegan los correos | En la instalación por defecto los correos van al Mailpit local (`http://localhost:8025`), no a internet. Configura SMTP real en **Administración → SMTP**. |
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

- **Bugs y feedback**: issues en [GitHub](https://github.com/va360labs/didacta-io/issues) — hay plantillas de bug, feedback y feature request.
- **Vulnerabilidades de seguridad**: sigue la política de [SECURITY.md](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md) — nunca las publiques en un issue.
