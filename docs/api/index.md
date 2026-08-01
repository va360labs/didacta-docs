# La API de Didacta

Toda la funcionalidad de Didacta se expone por una **API REST** (NestJS + Fastify). La web oficial es un cliente más: cualquier cosa que hace el panel puede hacerse por API.

## Lo esencial

| | |
| --- | --- |
| **URL base** | `https://tu-instancia/api/v1` |
| **Formato** | JSON (límite de body 16 MB; 50 MB para subir módulos ZIP) |
| **Autenticación** | JWT Bearer (sesiones de usuario) o **API keys** (`ApiKey lmsk_…`) — ver [Autenticación](autenticacion.md) |
| **Documentación interactiva** | Swagger UI en `/api/docs` · OpenAPI JSON en `/api/docs.json` |
| **Versionado** | `v1` en el path; sin negociación por header |

Fuera del prefijo `/api/v1` quedan las rutas operativas: `/healthz`, `/readyz`, `/livez` (probes), `/metrics` (Prometheus), `/api/docs` (Swagger), `/api/license` (estado de licencia) y `/scim/v2/*` (SCIM).

## Primer request

La forma más directa de integrarte es con una **API key** (se crean en el panel o vía `POST /api/v1/auth/api-keys`):

```bash
# Catálogo de cursos visible para integradores
curl -H "Authorization: ApiKey lmsk_xxx" \
  https://tu-instancia/api/v1/inscribe/courses
```

La API de inscripción (`/inscribe`) es la superficie pensada para integradores externos: dar de alta un alumno por email y matricularlo en cursos o grupos de acceso de forma **idempotente**:

```bash
curl -X POST https://tu-instancia/api/v1/inscribe \
  -H "Authorization: ApiKey lmsk_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alumno@example.com",
    "name": "Alumno de Ejemplo",
    "courseIds": ["..."],
    "accessGroupIds": ["..."]
  }'
```

## Cómo se organiza la superficie

| Bloque | Prefijo | Qué cubre |
| --- | --- | --- |
| Autenticación e identidad | `/auth`, `/me`, `/users` | Login, tokens, MFA, API keys, perfil, sesiones. |
| Setup | `/setup` | Primer arranque de la instancia (se cierra tras inicializar). |
| Administración | `/admin/*` | Organizaciones, usuarios, invitaciones, módulos, SMTP, SSO, SCIM, dominios, webhooks, IA. |
| Módulos funcionales | `/modules/<slug>/*` | Cursos, aprendizaje, comunidad, evaluaciones, certificados, pagos, Zoom, Fundae… |
| Integración externa | `/inscribe` | Alta y matrícula idempotente para sistemas externos. |
| Webhooks salientes | `/webhooks` | Gestión de endpoints suscritos a eventos de dominio. |
| Plataforma | `/branding`, `/storage`, `/audit`, `/tenant-settings` | Branding, ficheros, auditoría, ajustes de tenant. |

La [referencia de endpoints](referencia/index.md) recorre todos los bloques; para el detalle exhaustivo por endpoint (parámetros, schemas, respuestas), usa el **Swagger de tu propia instancia** (`/api/docs`) — siempre refleja la versión exacta que tienes desplegada.

## Siguiente paso

- [Autenticación](autenticacion.md) — JWT, API keys, scopes y SSO.
- [Convenciones](convenciones.md) — multi-tenancy, errores, rate limiting.
- [Referencia de endpoints](referencia/index.md) — el mapa completo de la superficie.
