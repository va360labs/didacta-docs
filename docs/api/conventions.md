# Convenciones de la API

## Multi-tenancy

Didacta es multi-tenant; cada petición opera en el contexto de una organización:

- **En las llamadas autenticadas, el tenant sale de la credencial** (claim `tenantId` del JWT o tenant de la API key) — no del dominio al que llamas.
- El **host** de la petición solo interviene donde aún no hay credencial: `GET /auth/tenant-context`, `signup`/`signin`/`forgot-password`, el `setup` inicial y los endpoints públicos del aula (`/public/*`, `/tenancy/*`, `/purchase-consent/*`, `/modules/billing/public/*`, `/membership/*`, `/modules/referrals/track`…). Se lee `X-Forwarded-Host` y, si no viene, `Host`. La resolución exige match **exacto** de dominio verificado (sin comodines de subdominio). Un dominio que no es de ningún aula activo responde `404` en los endpoints `/public/*` y `tenant: null` en `GET /auth/tenant-context`.
- Las llamadas con **credencial de provisioning** no tienen tenant: operan sobre la lista de organizaciones de la instancia.
- En base de datos, el aislamiento se refuerza con Row-Level Security por `tenant_id`.

### Estados del aula que cambian las respuestas

| Estado | Efecto en la API |
| --- | --- |
| Organización `SUSPENDED` o `ARCHIVED` | Sus sesiones dejan de valer (401) y su dominio deja de resolverse; `GET /tenancy/resolve` lo marca con `suspended: true`. |
| Puerta de suscripción `LOCKED` (plano de control) | **402** `SUBSCRIPTION_LOCKED` en casi toda la API. Siguen respondiendo el acceso (`/auth/*`), `/me/profile`, `/me/modules`, la marca (`/modules/theming/me` y logos), las sondas, `/api/license`, los webhooks entrantes de pasarelas y de Zoom y las rutas de organizaciones. |
| Altas congeladas | **409** `TENANT_SIGNUPS_FROZEN` en invitación individual, alta masiva, registro público y checkout de alumno nuevo. Quien ya está dentro no se ve afectado. |

## Idioma

- Los mensajes de error (`message`) de la API están en español; **programa siempre contra `code`**, que es estable. El panel los traduce por `code` (es/en/pt).
- Cada aula tiene un **idioma** (`es`, `en` o `pt`; `GET/PUT /admin/tenant-settings/academy`) que fija el de los correos y de lo que el servidor escribe a todos. Cada persona tiene además su `locale` de perfil (`es-ES`, `es-AR`, `en-US`, `pt-BR`; `PATCH /me/profile`).
- `GET /tenancy/locale` devuelve el idioma del aula del dominio de la petición. El checkout anónimo lee la cookie `didacta_locale` de la web para dejar el idioma del comprador en su cuenta, y `POST /setup/init` usa `Accept-Language` si no se indica idioma; fuera de eso, la API no negocia idioma por cabecera.

## Errores

La forma estándar es `{ "statusCode", "message", "code"? }` más campos extra según el caso. `code` es el campo para programar contra él.

| Situación | Código | Forma |
| --- | --- | --- |
| Validación de entrada (Zod) | 400 | `{ "message": "…", "issues": [{ "path", "message", "code" }] }` — `message` junta los problemas con « · » |
| Sin autenticar / token o clave inválidos | 401 | `AUTH_TOKEN_INVALID`, `AUTH_API_KEY_INVALID`, `AUTH_MISSING_BEARER_HEADER`, `session_revoked`, `account_suspended`… |
| Capability Enterprise no activa | **402** | `{ "statusCode": 402, "error": "CapabilityRequiredError", "code": "DIDACTA_CAPABILITY_REQUIRED", "capability": "feat:…", "path", "timestamp" }` |
| Aula en pausa (puerta de suscripción) | **402** | `{ "code": "SUBSCRIPTION_LOCKED", "message": "…" }` |
| Módulo desactivado para el tenant | 403 | `{ "code": "TENANT_MODULES_MODULE_NOT_ACTIVE", "detail": "<módulo>", … }` |
| Falta de permisos | 403 | `code` propio del recurso (`ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`, `AUTH_API_KEY_MISSING_SCOPES`…) · `mfa_required` si falta el segundo factor |
| Usuario sancionado | 403 | `{ "code": "user_restricted", "area", "reason", "expiresAt" }` — solo en escrituras del área sancionada |
| Conflictos, ya inicializado, altas congeladas | 409 | shape estándar |
| Rate limit general | 429 | `{ "code": "rate_limit_exceeded", "scope", "tier", "limit", "retryAfterSeconds" }` + header `Retry-After` |
| Rate limit de acceso | 429 | `{ "code": "AUTH_RATE_LIMITED", "retryAfterSeconds" }` |
| Errores de dominio | según el caso | `{ "statusCode", "code", "message", … }` — cada módulo mapea sus errores tipados (`code` estable) |

## Rate limiting

Hay **tres cubos** con ventana de 60 segundos y respaldo en Redis. Qué cubo te toca depende de cómo llegas:

| Cubo | Quién entra | Qué cuenta |
| --- | --- | --- |
| `user` | Una persona con sesión (JWT) | Sus peticiones, por usuario y aula — además cuentan en el tope del aula. |
| `authenticated` | Todo el tráfico de un aula: personas con sesión, **API keys, claves de agente y SCIM** | El tope del aula entera. Las keys y SCIM solo cuentan aquí. |
| `public` | Sin credencial | Por IP de cliente. |

Toda respuesta incluye:

```
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Tier
```

Límites por defecto (peticiones por minuto):

| Plan | `user` | `authenticated` (aula) | `public` (por IP) |
| --- | --- | --- | --- |
| Community | 100 | 100 | 30 |
| Enterprise <span class="didacta-chip didacta-chip--ee">EE</span> | 1000 | 1000 | 300 |

- El plan es **de la instancia**: lo da la capability `feat:api.rate_limit.elevated`. `GET /admin/rate-limit/info` (administrador) devuelve el plan activo y las cifras efectivas, también sin licencia, para poder mostrar la mejora.
- Se ajustan con `RATE_LIMIT_COMMUNITY_USER_PER_MIN`, `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN`, `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` (y las equivalentes `RATE_LIMIT_ENTERPRISE_*`) y `RATE_LIMIT_WINDOW_SECONDS`. Sin `RATE_LIMIT_COMMUNITY_USER_PER_MIN`, el límite por usuario es el del aula; los valores Enterprise nunca quedan por debajo de los Community.
- Si Redis no está disponible, el limitador **deja pasar** (no tumba la API) y las cabeceras dicen `Remaining` = `Limit`.
- Exentos: `/healthz`, `/readyz`, `/livez`, `/metrics`, `/api/docs` y `/api/license`.
- Con límites propios aparte: el [acceso](authentication.md#limite-de-intentos-de-acceso) (`AUTH_RATE_LIMITED`), la mensajería (20 mensajes/min y 30 señales de «escribiendo»/min por persona, `MESSAGING_RATE_LIMITED`) y el servidor MCP (ver [MCP](mcp.md)).

## Paginación

No hay un único formato; cada familia de endpoints usa el suyo:

| Dónde | Parámetros | Respuesta |
| --- | --- | --- |
| Listados de administración (`/admin/users`, grupos de acceso, grupos…) | `page`, `limit` (máx. 500 en usuarios) | `{ items, total, page, limit, hasMore }` |
| Ventas (`/admin/sales`, `/modules/payouts/sales`) | `page`, `pageSize` (máx. 100) | `{ items, page, pageSize, total, … }` con totales por divisa |
| Mensajería | `cursor` (50 por página, del más reciente al más antiguo) | Mensajes + cursor siguiente |
| Feed de la comunidad | `limit` y filtros | Lista |
| SCIM | `startIndex` (base 1) y `count` | `ListResponse` de SCIM |

## Gating por edición y por módulo

- **Endpoint Enterprise sin licencia** → `402` con la capability requerida en el cuerpo. La lista de capabilities está en [Enterprise → Capabilities](../enterprise/capabilities.md).
- **Endpoint de un módulo desactivado** para el tenant → `403` `TENANT_MODULES_MODULE_NOT_ACTIVE` («El módulo X no está activo para este tenant»). Vale para todo lo que cuelga de `/modules/<slug>/…`.
- **Rutas que no existen sin configuración del operador** → `404`: los cobros de plataforma (`/modules/payouts/*`, necesitan el modo plataforma) y la subida de vídeo (`/video/uploads/*`, necesita Bunny Stream). Son cosas de Didacta Cloud; ver la [referencia](reference/index.md).

## Webhooks salientes

Suscríbete a eventos de dominio con endpoints HTTPS propios (**`/api/v1/webhooks/endpoints`**, rol admin). Cada entrega es un `POST` JSON con `event`, `data`, `occurredAt`, `tenantId`, `deliveryId` y el objeto `learner` (la persona detrás del evento, o `null`). Cabeceras: `X-Didacta-Event`, `X-Didacta-Tenant`, `X-Didacta-Attempt` y `X-Didacta-Delivery` — **deduplica por `X-Didacta-Delivery`**. El secret del endpoint se devuelve **una sola vez** al crearlo.

- **Community**: 1 endpoint por organización, máx. 3 tipos de evento (`["*"]` cuenta como uno), entrega directa síncrona con 5 s de timeout y 1 reintento, **sin firma**.
- **Enterprise** <span class="didacta-chip didacta-chip--ee">EE</span> (`feat:api.webhooks.high_throughput`): 20 endpoints, eventos ilimitados, cola con backoff exponencial, **firma HMAC-SHA256** (`X-Didacta-Signature`) y dead-letter con reintento desde el panel.

El catálogo completo de eventos, el formato exacto y la verificación de la firma están en [Webhooks](webhooks.md).

## Otras convenciones

- **Trazabilidad**: puedes enviar `x-trace-id`; si no, se genera y se propaga en los logs.
- **Cabeceras de seguridad**: toda respuesta lleva `X-Content-Type-Options: nosniff` y `Referrer-Policy: no-referrer`; las de la API añaden `Content-Security-Policy` restrictiva y `X-Frame-Options: DENY`, y `Strict-Transport-Security` cuando llega por HTTPS (se apaga con `WEB_HSTS=off`). Los ficheros subidos (`/storage/file/*`) llevan su propia CSP `sandbox`.
- **CORS no está habilitado**: la API está pensada para consumirse mismo-origen (la web usa rewrites) o servidor-a-servidor. Un cliente de navegador desde otro origen necesitaría un proxy propio.
- **Idempotencia** donde importa: `/inscribe`, `/integrations/orders` (por `source` + `reference`), webhooks de Stripe, activación de módulos y el alta en 3 preguntas del modo sencillo (`quick-start`) son seguros de reintentar.
- **Importes**: siempre en la unidad mínima de la divisa (`amountCents`) y con la divisa explícita; nunca se suman divisas distintas.
