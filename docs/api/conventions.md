# Convenciones de la API

## Multi-tenancy

Didacta es multi-tenant; cada petición opera en el contexto de una organización:

- **En las llamadas autenticadas, el tenant sale del token** (claim `tenantId` del JWT o tenant de la API key) — no del dominio al que llamas.
- El **host** de la petición solo interviene donde aún no hay credencial: `GET /auth/tenant-context` (qué organización corresponde a este dominio), `signup`/`signin`/`forgot-password` y el `setup` inicial. La resolución exige match **exacto** de dominio verificado (sin comodines de subdominio).
- En base de datos, el aislamiento se refuerza con Row-Level Security por `tenant_id`.

## Errores

| Situación | Código | Forma |
| --- | --- | --- |
| Validación de entrada (Zod) | 400 | `{ "message": "…", "issues": [{ "path", "message", "code" }] }` |
| Sin autenticar / licencia inválida | 401 | shape estándar |
| Capability Enterprise no activa | **402** | `{ "statusCode": 402, "error": "CapabilityRequiredError", "capability": "feat:…", "path", "timestamp" }` |
| Módulo desactivado para el tenant / permisos | 403 | shape estándar |
| Ya inicializado, conflictos | 409 | shape estándar |
| Rate limit | 429 | `{ "code": "rate_limit_exceeded", "tier", "limit", "retryAfterSeconds" }` + header `Retry-After` |
| Errores de dominio | según el caso | `{ "statusCode", "code", "message", … }` — cada módulo mapea sus errores tipados (`code` estable para programar contra él) |

## Rate limiting

Global, por organización (o `anonymous`), con ventana de 60 segundos y respaldo en Redis. Toda respuesta incluye:

```
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Tier
```

Límites por defecto: **100 req/min** autenticado y **30 req/min** anónimo en Community (configurables con `RATE_LIMIT_COMMUNITY_*`); el tier Enterprise (`feat:api.rate_limit.elevated`) sube a 1000/300. Exentos: probes, `/metrics`, `/api/docs`, `/api/license`.

## Gating por edición y por módulo

- **Endpoint Enterprise sin licencia** → `402` con la capability requerida en el cuerpo. La lista de capabilities está en [Enterprise → Capabilities](../enterprise/capabilities.md).
- **Endpoint de un módulo desactivado** para el tenant → `403` («El módulo X no está activo para este tenant»).

## Webhooks salientes

Suscríbete a eventos de dominio con endpoints HTTPS propios (**`/api/v1/webhooks/endpoints`**, rol admin):

- Cuerpo del `POST`:

  ```json
  {
    "event": "learning.enrollment.created",
    "data": { "enrollmentId": "…", "userId": "…", "courseId": "…", "source": "API" },
    "occurredAt": "2026-08-18T17:24:05.412Z",
    "tenantId": "…",
    "deliveryId": "wd_9f0c…",
    "learner": {
      "id": "…",
      "email": "ana@ejemplo.com",
      "name": "Ana Ruiz",
      "externalSource": "learndash",
      "externalId": "4471"
    }
  }
  ```

- **`learner` es quien te ahorra el problema de identidad**: `data` solo trae `userId`, y no hay endpoint que traduzca un `userId` a una persona (todo `/integrations` busca por email). Llega `null` cuando el evento no tiene persona detrás (p. ej. `subscriptions.invoice.refunded`).
- `occurredAt` es cuándo **ocurrió** el evento, no cuándo se envió: tras una reentrega del outbox pueden diferir en horas.
- Catálogo (solo eventos que alguien publica de verdad): `*`, `learning.enrollment.created`, `learning.enrollment.paused`, `learning.enrollment.resumed`, `learning.enrollment.cancelled`, `learning.course.completed`, `community.post.created`, `community.comment.created`, `billing.order.created`, `billing.order.completed`, `billing.order.failed`, `billing.order.refunded`, `subscriptions.subscription.created`, `subscriptions.subscription.activated`, `subscriptions.subscription.past_due`, `subscriptions.subscription.unpaid`, `subscriptions.subscription.canceled`, `subscriptions.invoice.paid`, `subscriptions.invoice.payment_failed`, `subscriptions.invoice.refunded`, `subscriptions.membership.activated`, `fundae.group.closed`, `assessments.attempt.submitted` — con suscripción por prefijo (`billing.*`).
- Cabeceras: `X-Didacta-Event`, `X-Didacta-Tenant`, `X-Didacta-Attempt` y `X-Didacta-Delivery`. **Deduplica por `X-Didacta-Delivery`**: es estable entre reintentos y entre reentregas del mismo evento.
- El secret del endpoint se devuelve **una sola vez** al crearlo.
- **Community**: 1 endpoint por organización, máx. 3 tipos de evento —`["*"]` cuenta como uno y te suscribe a todo el catálogo—, entrega directa con 1 reintento y **sin firma**. Si necesitas autenticar el aviso en community, mete un token en la propia URL. **Enterprise** (`feat:api.webhooks.high_throughput`): 20 endpoints, eventos ilimitados, cola con backoff exponencial, **firma HMAC-SHA256** (`X-Didacta-Signature`) y dead-letter con reintento desde el panel.
- La entrega en community es **síncrona** dentro del despacho del evento, con 5 s de timeout: responde `2xx` en cuanto recibas y procesa después.

## Otras convenciones

- **Paginación** en listados de administración: `{ items, total, page, limit, hasMore }`.
- **Trazabilidad**: puedes enviar `x-trace-id`; si no, se genera y se propaga en los logs.
- **CORS no está habilitado**: la API está pensada para consumirse mismo-origen (la web usa rewrites) o servidor-a-servidor. Un cliente de navegador desde otro origen necesitaría un proxy propio.
- **Idempotencia** donde importa: `/inscribe`, webhooks de Stripe y activación de módulos son seguros de reintentar.
