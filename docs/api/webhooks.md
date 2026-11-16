# Webhooks

Didacta usa webhooks en los dos sentidos:

| | Quién llama a quién | Para qué | Dónde se configura |
|---|---|---|---|
| **[Entrantes](#webhooks-entrantes)** | Stripe, Zoom, WooCommerce → Didacta | Que el aula se entere de un pago, de una clase que empieza o de un pedido | En el proveedor, apuntando al dominio de tu aula |
| **[Salientes](#webhooks-salientes)** | Didacta → tu sistema | Que tu CRM, tu tienda o tu n8n se enteren de lo que pasa en el aula | **Administración → Webhooks** o `/api/v1/webhooks/endpoints` |

!!! info "Una página de referencia"
    Esta página lista los nombres exactos de rutas, cabeceras, eventos y límites. Para el caso de uso «mi tienda vive fuera», empieza por [Vender e integrar desde fuera](external-integrations.md).

---

## Webhooks entrantes

### Reglas comunes

- **La URL cuelga del dominio de tu aula** (`https://tu-aula.example.com/api/v1/…`). Didacta mira el `Host` de la petición **antes de leer ningún secreto** para saber de qué aula es. Por eso cada aula apunta el webhook de su proveedor a **su propio dominio verificado**; en una instalación con varias aulas, cada una tiene su URL y su secreto.
- **Sin firma válida no se procesa nada**: son rutas públicas (el proveedor no manda sesión) y la autenticación es la firma del cuerpo crudo con el secreto de **esa aula**.
- **El aula que firma es el aula para la que se procesa.** Si un evento firmado nombra por dentro a otra aula (por `metadata`, ids de pedido o de reunión), se rechaza con `403` y queda rastro en auditoría.
- **Idempotentes**: cada evento se guarda antes de procesarse, con su identificador como clave única. Una reentrega no duplica nada, así que el proveedor puede reintentar sin riesgo.

### Stripe

Hay dos rutas, una por módulo. Cada una verifica la cabecera `stripe-signature` con su secreto (**Administración → Configuración → Pagos**, `/admin/settings?tab=payments`; si el aula no tiene el suyo, se usa el de la instancia — ver [Variables de entorno](../configuration/environment-variables.md)).

| Ruta | Módulo | Secreto |
|---|---|---|
| `POST /api/v1/modules/billing/webhook` | Compra de curso suelto ([Stripe Checkout](../modules/catalog/billing.md)) | «Secreto del webhook» |
| `POST /api/v1/modules/subscriptions/webhook` | Suscripciones y membresía ([Suscripciones](../modules/catalog/subscriptions.md)) | «Secreto del webhook de suscripciones» (opcional: si lo dejas vacío, se usa el de arriba) |

Eventos de Stripe que tienes que activar en cada endpoint:

| Endpoint | Eventos |
|---|---|
| `…/billing/webhook` | `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`, `charge.refunded` |
| `…/subscriptions/webhook` | `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`, `charge.refunded`, `checkout.session.completed` (alta de membresía), `checkout.session.expired` |

!!! tip "Un solo endpoint puede servir a los dos módulos"
    La ruta de suscripciones reparte cada evento ya verificado a suscripciones, membresía y, al final, a compra de curso suelto, y cada módulo procesa solo lo suyo. Si vendes cursos sueltos **y** suscripciones con la misma cuenta de Stripe, puedes registrar **un** endpoint (`…/subscriptions/webhook`) con la **unión** de los eventos de la tabla.

!!! warning "Sin estos eventos, se cobra pero no se entrega"
    El acceso se concede al recibir `checkout.session.completed` (o `async_payment_succeeded` en los métodos de pago diferidos) y se retira con `charge.refunded` o `customer.subscription.deleted`. Si no los activas, el cobro entra en Stripe y el alumno no recibe el curso, o el reembolso no le quita el acceso.

Respuestas útiles al depurar:

| HTTP | Significado |
|---|---|
| `200` | Recibido y procesado (o archivado sin efecto si el evento no es de este módulo) |
| `401` | Falta `stripe-signature`, la firma no es válida o Stripe no está configurado en esta aula |
| `404` | El dominio de la petición no corresponde a ningún aula |
| `403` | El evento firmado nombra a otra aula |

Stripe espera un `2xx` rápido y, si no lo recibe, reintenta durante días. Como Didacta guarda el evento antes de procesarlo, un reintento no duplica nada.

!!! note "Modo plataforma"
    Si el operador de la instalación activa el modo plataforma de cobros (variable `PAYMENTS_PLATFORM_STRIPE_SECRET_KEY`), el webhook de Stripe es **uno solo** para toda la instalación (`POST /api/v1/modules/payouts/webhook`, con los secretos del operador): las aulas no configuran nada y el aula se decide por el registro de objetos de la plataforma, no por el dominio.

### Zoom

`POST /api/v1/webhooks/zoom` — en la app de Zoom, usa como URL de eventos **la de tu aula**: `https://tu-aula.example.com/api/v1/webhooks/zoom`.

1. En **Administración → Zoom** guarda el **Webhook Secret Token** de tu app de Zoom. Es **por aula** (se guarda cifrado junto a las credenciales Server-to-Server). Guárdalo **antes** de pulsar «Validate» en Zoom: la validación va firmada con ese token y, sin él, Didacta responde `401`.
2. En Zoom, suscribe la app a los eventos de reunión y de grabación.

| | |
|---|---|
| **Firma** | `x-zm-signature: v0=<hmac-sha256 hex>` sobre `v0:<x-zm-request-timestamp>:<cuerpo crudo>`; el timestamp no puede desviarse más de 5 minutos |
| **Handshake** | Responde al evento `endpoint.url_validation` con `plainToken` y `encryptedToken` |
| **Eventos que cambian algo** | `meeting.started` (clase en directo), `meeting.ended` (terminada; dispara la encuesta post-clase si está activa) y `recording.completed` (adjunta la grabación) |
| **Cualquier otro evento** | `200` con `{"result":"IGNORED"}`, para que Zoom no reintente |
| **Idempotencia** | Por una clave derivada del contenido del aviso (Zoom no manda `event_id`) |

Varias aulas del mismo dueño pueden compartir **una** app de Zoom y una URL **si usan exactamente el mismo Secret Token**; en cualquier otro caso un aviso sobre una reunión de otra aula se rechaza con `403`.

El registro de las entregas está en **Administración → Webhooks → pestaña Zoom**.

### WooCommerce

`POST /api/v1/modules/payment-connections/woo-webhook?tenant=<slug-del-aula>` — lo llama tu tienda WooCommerce. A diferencia de los anteriores, el aula viaja en la query (`?tenant=`), porque WooCommerce no sabe de multi-aula.

| | |
|---|---|
| **Firma** | `x-wc-webhook-signature`: HMAC-SHA256 del cuerpo crudo con el secreto del webhook, en **base64** (no hex) y sin prefijo |
| **Temas** | `order.created`, `order.updated`, `order.deleted`. Otros temas: `200` ignorado, para que WooCommerce no desactive el webhook |
| **Ping de verificación** | El `POST` de prueba sin firma que hace WooCommerce al guardarlo recibe `200` y no se procesa |
| **Efecto** | **Refleja el pedido** en la ficha de la persona; no reparte accesos |
| **Pedido sin email** | `200` con `skipped`, sin reintentos |

Ver [Conexiones de pago](../modules/catalog/payment-connections.md).

### Vídeo gestionado (Bunny Stream)

`POST /api/v1/video/bunny/webhook` lo llama el servicio de vídeo gestionado cuando cambia el estado de un vídeo (firma `x-bunnystream-signature`). Lo configura el operador de la instalación, no el aula.

---

## Webhooks salientes

Los webhooks salientes avisan a tu sistema —por HTTPS, con un `POST` JSON— cada vez que ocurre un evento de dominio en el aula: una matrícula, un pago, una renovación, un curso terminado.

### Configurarlos

En **Administración → Webhooks → Salientes** (`/admin/webhooks`), o por API con rol de administrador (`tenant_admin` o `super_admin`):

| Método | Ruta | Para qué |
|---|---|---|
| `GET` | `/api/v1/webhooks/info` | Tier activo, límites efectivos y catálogo de eventos suscribibles |
| `GET` | `/api/v1/webhooks/endpoints[/:id]` | Lista / detalle (el secreto sale enmascarado) |
| `POST` | `/api/v1/webhooks/endpoints` | Crea un endpoint |
| `PUT` | `/api/v1/webhooks/endpoints/:id` | Edita; enviar `secret` lo rota |
| `DELETE` | `/api/v1/webhooks/endpoints/:id` | Elimina (`204`, idempotente) |

```bash
curl -X POST https://tu-aula.example.com/api/v1/webhooks/endpoints \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{
        "url": "https://crm.example.com/hooks/didacta",
        "eventTypes": ["learning.enrollment.created", "billing.order.*"]
      }'
```

- `url`: **HTTPS** obligatorio (solo se admite `http://` para `localhost` y `127.0.0.1`, en desarrollo). Una misma URL no puede repetirse en el aula (`409`).
- `eventTypes`: al menos uno, de la lista de abajo. `["*"]` los recibe **todos**; un prefijo con comodín (`billing.*`, `learning.*`) recibe todo lo de ese prefijo. Los nombres no se validan contra el catálogo: uno que no exista no da error, pero nunca llegará.
- `secret`: opcional (16–256 caracteres). Si no lo mandas, Didacta genera uno (`whsec_…`). **Se devuelve en claro una sola vez**, al crear o al rotar; después solo se ve enmascarado.
- `active`: `false` pausa el endpoint sin borrarlo.

### Catálogo de eventos

Estos son **todos** los eventos a los que puedes suscribirte; cada uno tiene detrás un `publish()` real. (Los módulos emiten más eventos entre sí, pero solo estos se reenvían fuera; ver [Eventos y hooks](../modules/building-a-module/events-and-hooks.md).) Dentro de `data` van, además de lo que se indica, los campos que cada módulo añada en el futuro: tolera campos nuevos.

| Evento | Cuándo | Campos principales de `data` |
|---|---|---|
| `learning.enrollment.created` | Alguien queda matriculado, por la vía que sea | `enrollmentId`, `userId`, `courseId`, `source` |
| `learning.enrollment.paused` | Matrícula suspendida (p. ej. por impago): conserva el progreso, pierde el acceso | `courseId`, `userId` |
| `learning.enrollment.resumed` | Recupera el acceso | `courseId`, `userId` |
| `learning.enrollment.cancelled` | Baja de la matrícula (reembolso, cancelación, salida de un grupo) | `courseId`, `userId` (o `enrollmentId`) |
| `learning.course.completed` | Termina el curso | `enrollmentId`, `courseId`, `userId` |
| `community.post.created` | Nueva publicación | `postId`, `authorId`, `courseId` |
| `community.comment.created` | Nuevo comentario | `commentId`, `postId`, `authorId`, `parentCommentId` |
| `billing.order.created` | Se inició la compra de un curso suelto (aún sin pagar) | `orderId`, `productId`, `courseId`, `userId`, `stripeSessionId`, `amount`, `currency` |
| `billing.order.completed` | Compra pagada | `orderId`, `productId`, `courseId`, `userId`, `userCreated`, `amountPaid`, `currency`, `customerEmail`, `attribution` |
| `billing.order.failed` | El pago se cayó o el diferido no llegó a cobrarse | `orderId`, `productId`, `courseId`, `userId` |
| `billing.order.refunded` | Compra reembolsada | `orderId`, `productId`, `courseId`, `userId`, `amountRefunded` |
| `subscriptions.subscription.created` | Se inició una suscripción a un curso | `subscriptionId`, `courseId`, `userId`, `stripeSessionId`, `priceId`, `interval` |
| `subscriptions.subscription.activated` | Suscripción activa (alta o recuperación tras un impago) | `subscriptionId`, `courseId`, `planId`, `userId`, `currentPeriodEnd`, `recovery` |
| `subscriptions.subscription.past_due` | Falló un cobro y sigue dentro (periodo de gracia): la ventana para recuperarlo | `subscriptionId`, `courseId`, `planId`, `userId`, `gracePeriodEndsAt` |
| `subscriptions.subscription.unpaid` | Se agotó la gracia sin cobrar | `subscriptionId`, `courseId`, `planId`, `userId` |
| `subscriptions.subscription.canceled` | Fin de la suscripción | `subscriptionId`, `courseId`, `planId`, `userId`, `immediate` |
| `subscriptions.invoice.paid` | **Renovación cobrada**: el único evento que distingue «sigue pagando» de «se dio de alta» | `subscriptionId`, `stripeInvoiceId`, `amount`, `currency` |
| `subscriptions.invoice.payment_failed` | Falló el cobro de la cuota | `subscriptionId`, `stripeInvoiceId`, `attemptCount`, `nextPaymentAttempt` |
| `subscriptions.invoice.refunded` | Cuota reembolsada (se publica **sin persona**: `learner: null`) | `subscriptionId`, `stripeInvoiceId`, `amountRefunded`, `currency` |
| `subscriptions.membership.activated` | Alta de una membresía | `subscriptionId`, `planId`, `userId`, `userCreated`, `stripeSubscriptionId`, `attribution` |
| `fundae.group.closed` | Se cierra un grupo Fundae | `groupId`, `companyId`, `creditoConsumidoCents` |
| `assessments.attempt.submitted` | Se entrega un intento de examen | `attemptId`, `quizId`, `userId`, `enrollmentId`, `lessonId`, `scoreEarned`, `scoreMax`, `scorePercent`, `passed`, `needsReview` |

!!! note "Eventos que se publican pero no se reenvían"
    Entre otros, `billing.order.partially_refunded`, `subscriptions.invoice.partially_refunded` o `subscriptions.subscription.paused` **no** están en el catálogo saliente de esta versión. Si los necesitas, usa `learning.enrollment.*` para saber si el acceso cambió, o pide añadirlos.

### El cuerpo

```json
{
  "event": "learning.enrollment.created",
  "data": {
    "enrollmentId": "…",
    "userId": "…",
    "courseId": "…",
    "source": "API"
  },
  "occurredAt": "2026-11-16T17:24:05.412Z",
  "tenantId": "…",
  "deliveryId": "wd_9f0c…",
  "learner": {
    "id": "…",
    "email": "ana@example.com",
    "name": "Ana Ruiz",
    "externalSource": null,
    "externalId": null
  }
}
```

| Campo | Qué es |
|---|---|
| `event` | Nombre del evento |
| `data` | Datos del evento (tabla de arriba) |
| `occurredAt` | Cuándo **ocurrió** el evento (ISO 8601), no cuándo se envió: tras una reentrega pueden diferir en horas |
| `tenantId` | El aula |
| `deliveryId` | Identificador de la entrega, **estable** entre reintentos y reentregas del mismo evento: tu clave de deduplicación |
| `learner` | La persona a la que se refiere el evento: `id`, `email`, `name`, `externalSource` y `externalId` (la clave con la que el sistema de origen conoce a esa misma persona, en aulas migradas). **`null`** si el evento no tiene persona detrás (`subscriptions.invoice.refunded`) |
| `attempt` | Solo en Enterprise: número de intento de la cola |

`data` solo trae `userId` y no hay endpoint que traduzca un `userId` a una persona: por eso llega `learner` ya resuelto. El sujeto es el `userId` del propio evento y, si no lo tiene, el actor que lo provocó.

### Cabeceras

| Cabecera | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Didacta-Event` | Nombre del evento |
| `X-Didacta-Delivery` | El `deliveryId`. **Deduplica por esta cabecera** |
| `X-Didacta-Attempt` | Número de intento |
| `X-Didacta-Tenant` | El `tenantId` |
| `X-Didacta-Signature` | `sha256=<hex>`. **Solo Enterprise** (ver abajo) |

### Community, Enterprise y Cloud

El comportamiento depende de si la instalación tiene la capability **`feat:api.webhooks.high_throughput`** (ver [Capabilities](../enterprise/capabilities.md)). Se evalúa en cada envío: cambiar de licencia surte efecto sin reiniciar.

| | Community | Enterprise (`feat:api.webhooks.high_throughput`) |
|---|---|---|
| Endpoints por aula | 1 (`WEBHOOKS_COMMUNITY_MAX_ENDPOINTS`) | 20 (`WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS`) |
| Tipos de evento por endpoint | 3 (`WEBHOOKS_COMMUNITY_MAX_EVENTS`); `["*"]` cuenta como uno y recibe todo el catálogo | Sin límite |
| Entrega | **Directa y síncrona** dentro del despacho del evento | Cola dedicada (`didacta.webhooks.outbound`, BullMQ + Redis) |
| Reintentos | 1 reintento inmediato (a los 250 ms), 2 intentos en total | 5 reintentos con espera creciente: **30 s, 2 min, 8 min, 30 min, 2 h** (6 intentos en total) |
| Firma | **Sin firma** | **HMAC-SHA256** en `X-Didacta-Signature` |
| Si se agotan los reintentos | El evento se pierde (solo queda un aviso en el log del servidor) | Pasa a **dead-letter**: se puede reintentar o descartar desde el panel o la API |
| Métricas | — | Prometheus: `webhooks_outbound_total`, `webhooks_outbound_duration_seconds`, `webhooks_outbound_attempts`, `webhooks_outbound_dead_letter_total`, `webhooks_outbound_queue_depth` |

**Didacta Cloud**: la edición `cloud` tiene su propia licencia, y el aula aplica la columna Enterprise **si esa licencia incluye** `feat:api.webhooks.high_throughput`; si no, la de Community. Los límites no dependen del nombre de la edición, sino de esa capability.

En todos los casos:

- El timeout de cada `POST` es de **5 s** (`WEBHOOKS_TIMEOUT_MS`). Cualquier respuesta que no sea `2xx` cuenta como fallo. **Responde `2xx` en cuanto recibas** y procesa después.
- Los avisos son **best-effort**: un endpoint caído nunca frena el proceso de negocio ni a los demás endpoints.
- Una entrega puede llegar **más de una vez**. Deduplica por `X-Didacta-Delivery`.

!!! warning "En Community el `POST` no va firmado"
    Si necesitas asegurarte de que el aviso viene de tu aula, mete un **token secreto en la propia URL** del endpoint (`https://crm.example.com/hooks/didacta?token=…`) y compruébalo, o filtra por la IP de tu instalación.

### Verificar la firma (Enterprise)

La firma es `sha256=` + el HMAC-SHA256 en hexadecimal del **cuerpo exacto** que recibes (los bytes crudos, antes de parsear el JSON), con el `secret` del endpoint como clave.

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

function isFromDidacta(rawBody, signatureHeader, secret) {
  const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex');
  const a = Buffer.from(signatureHeader ?? '');
  const b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

!!! warning "Excepción: el respaldo sin Redis"
    Si Didacta no consigue encolar la entrega (por ejemplo, Redis caído en ese instante), la envía de forma directa, como en Community, para no perder el evento: **esa entrega no lleva `X-Didacta-Signature`**. Si tu receptor rechaza lo que no viene firmado, perderás esos avisos; en su lugar, acepta los no firmados solo si traen tu token en la URL.

### Dead-letter (Enterprise)

Las entregas que agotan los reintentos se guardan para revisarlas. Están en **Administración → Webhooks** y por API (rol de administrador; sin licencia responden `402`):

| Método | Ruta | Para qué |
|---|---|---|
| `GET` | `/api/v1/admin/webhooks/dead-letter?limit=` | Lista (de 1 a 200; 50 por defecto) con `eventType`, `payload`, `lastError`, `attempts` |
| `POST` | `/api/v1/admin/webhooks/dead-letter/:id/retry` | Reencola la entrega (`202`) y borra la fila si el reencolado funcionó |
| `DELETE` | `/api/v1/admin/webhooks/dead-letter/:id` | La descarta (`204`) |

!!! note "Una entrega reencolada llega más pobre"
    Se reconstruye a partir del evento y su `data`: llega con `learner: null` y un `deliveryId` que termina en `-legacy`, no con el original. Si tu receptor deduplica por `deliveryId`, no la tomará por un duplicado.

### Variables de entorno

| Variable | Por defecto | Qué controla |
|---|---|---|
| `WEBHOOKS_TIMEOUT_MS` | `5000` | Timeout del `POST` a un endpoint |
| `WEBHOOKS_COMMUNITY_MAX_ENDPOINTS` | `1` | Endpoints por aula en Community |
| `WEBHOOKS_COMMUNITY_MAX_EVENTS` | `3` | Tipos de evento por endpoint en Community |
| `WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS` | `20` | Endpoints por aula con la capability |

La cola de Enterprise requiere `REDIS_URL`.

## Ver también

- [Vender e integrar desde fuera](external-integrations.md) — qué eventos usar para una tienda externa.
- [Eventos y hooks](../modules/building-a-module/events-and-hooks.md) — el catálogo completo de eventos de dominio que cada módulo declara.
- [Convenciones](conventions.md) — errores, multi-tenancy y límites de peticiones.
- [Capabilities de Enterprise](../enterprise/capabilities.md).
