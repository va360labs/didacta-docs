# La API de Didacta

Toda la funcionalidad de Didacta se expone por una **API REST** (NestJS + Fastify). La web oficial es un cliente más: cualquier cosa que hace el panel puede hacerse por API.

## Lo esencial

| | |
| --- | --- |
| **URL base** | `https://tu-instancia/api/v1` |
| **Formato** | JSON (límite de body 16 MB; 50 MB para subir módulos ZIP) |
| **Autenticación** | JWT Bearer (sesiones de usuario), **API keys** (`ApiKey lmsk_…`), credencial de provisioning (`Provisioning didprov_…`) y claves de agente para MCP — ver [Autenticación](authentication.md) |
| **Documentación interactiva** | Swagger UI en `/api/docs` · OpenAPI JSON en `/api/docs.json` |
| **Versionado** | `v1` en el path; sin negociación por header |

Fuera del prefijo `/api/v1` quedan las rutas operativas: `/healthz`, `/readyz`, `/livez` (probes), `/metrics` (Prometheus), `/api/docs` (Swagger), `/api/license` (estado público de la licencia) y `/scim/v2/*` (SCIM).

## Primer request

La forma más directa de integrarte es con una **API key** (la crea un administrador en el panel o con `POST /api/v1/auth/api-keys`):

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
| Autenticación e identidad | `/auth`, `/me`, `/users` | Login, tokens, MFA, API keys, perfil, sesiones, acceso de soporte. |
| Setup | `/setup` | Primer arranque de la instancia (se cierra tras inicializar). |
| Sitio público del aula | `/public/*`, `/tenancy/*`, `/purchase-consent/*` | Portada pública, SEO, quién vende, resolución de dominio e idioma. Sin sesión. |
| Administración del aula | `/admin/*` | Usuarios, invitaciones, módulos, SMTP, Stripe, SSO, SCIM, dominios, webhooks, IA, modo sencillo, ventas, vídeo, moderación. |
| Instancia y plano de control | `/admin/tenants`, `/admin/provisioning-credential`, `/admin/license`, `/admin/instance-settings` | Organizaciones, credencial de provisioning, accesos de soporte, puerta de suscripción, licencia. Ver [Instancia y plano de control](reference/instance-operations.md). |
| Módulos funcionales | `/modules/<slug>/*` | Cursos, aprendizaje, comunidad, evaluaciones, certificados, pagos, cobros, retención, soporte, Zoom, Fundae… |
| Integración externa | `/inscribe`, `/integrations`, `/community-api` | Alta y matrícula idempotente, lectura de fichas y compras, publicación en la comunidad. |
| Agentes de IA | `/agent/keys`, `/mcp` | Claves de agente y servidor MCP del aula — ver [MCP](mcp.md). |
| Webhooks salientes | `/webhooks`, `/admin/webhooks` | Endpoints suscritos a eventos de dominio — ver [Webhooks](webhooks.md). |
| Plataforma | `/branding`, `/storage`, `/audit`, `/tenant-settings`, `/video` | Branding, ficheros, auditoría, ajustes de tenant, vídeo. |

La [referencia de endpoints](reference/index.md) recorre todos los bloques; para el detalle exhaustivo por endpoint (parámetros, schemas, respuestas), usa el **Swagger de tu propia instancia** (`/api/docs`) — siempre refleja la versión exacta que tienes desplegada.

!!! note "Marcas de edición"
    En la referencia, <span class="didacta-chip didacta-chip--ee">EE</span> indica que el endpoint exige una capability Enterprise (sin licencia responde **402**). **Cloud** indica algo que solo existe cuando el operador lo enciende —el modo plataforma de cobros o la subida de vídeo de Didacta Cloud—; en una instalación autoalojada sin esa configuración esas rutas no existen.

## Siguiente paso

- [Autenticación](authentication.md) — JWT, API keys y scopes, credencial de provisioning, SSO.
- [Convenciones](conventions.md) — multi-tenancy, errores, rate limiting, paginación.
- [Referencia de endpoints](reference/index.md) — el mapa completo de la superficie.
