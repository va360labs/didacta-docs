# Referencia — Pagos, aula virtual e IA

Billing, suscripciones, conexiones de pago, Zoom y los tres módulos de IA. Todas las rutas cuelgan de `/api/v1`.

## Pago único (billing) — `/modules/billing`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/billing/offer/:courseId` | Bearer | Oferta del curso (`forSale`, precio, descuento). Tolerante: sin Stripe configurado devuelve `forSale: false`, nunca 500. |
| POST | `/modules/billing/checkout/:courseId` | Bearer | Crea la Checkout Session de Stripe y devuelve la URL. 409 si el curso no está publicado o ya tienes acceso. |
| GET · POST · PATCH · DELETE | `/modules/billing/products[/:id]` | admin | Productos (curso ↔ precio de Stripe): alta con `stripePriceId` existente **o** `amountCents` (crea Product+Price), activación, borrado (no toca órdenes históricas). |
| GET | `/modules/billing/public/catalog` · `public/offer/:courseId` | Público (tenant por dominio) | Catálogo público de venta · oferta pública. Degradan a vacío sin Stripe. La oferta **no filtra existencia**: curso inexistente o sin publicar responde 200 `{ forSale: false, options: [] }`, nunca 404. |
| POST | `/modules/billing/public/checkout/:courseId` | Público | **Checkout anónimo**: el visitante compra sin cuenta. Body `{ optionId?, email? }` — el `email` solo **precarga** el formulario de Stripe; la cuenta se crea SIEMPRE con el email confirmado en el pago. 404 curso inexistente (o `:courseId` no UUID) · 409 no publicado · 503 sin pasarela. |
| POST | `/modules/billing/webhook` | Firma Stripe (`stripe-signature`) | Recibe `checkout.session.completed/expired`, `charge.refunded`. Idempotente por `stripe_event_id`. |

## Suscripciones — `/modules/subscriptions`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/subscriptions/checkout/:courseId` | Bearer | Checkout `mode=subscription`: `{ stripePriceId }` (recurring). |
| GET | `/modules/subscriptions/me` | Bearer | Mis suscripciones (incluye canceladas, con plan y curso). Devuelve `[]` si el módulo no está configurado. |
| GET | `/modules/subscriptions/me/:id/invoices` | Bearer (propietario) | Facturas con URL hosted de Stripe. |
| POST | `/modules/subscriptions/me/:id/cancel` | Bearer (propietario) | Cancela: `{ immediate? }` (por defecto al final del periodo). |
| POST | `/modules/subscriptions/me/membership/pay-now` | Bearer | Termina el trial de la membresía y cobra ya. |
| POST | `/modules/subscriptions/webhook` | Firma Stripe (secret propio) | `customer.subscription.*`, `invoice.*`; fulfillment de membresía idempotente. |
| POST | `/modules/subscriptions/admin/grace-expiration/run-now` | super_admin | Fuerza la expiración de periodos de gracia (QA) → 202. |

Errores comunes: `SUBSCRIPTIONS_PRICE_NOT_RECURRING` 422 · `SUBSCRIPTIONS_ALREADY_ACTIVE` 409 · `*_STRIPE_CONFIG_MISSING` 503 · `*_STRIPE_API_ERROR` 502.

## Conexiones de pago — `/modules/payment-connections` (super_admin)

**Conexiones** (solo lectura sobre las cuentas):

| Método | Ruta | Qué hace |
|---|---|---|
| POST · GET | `…/connections` | Conecta cuenta (Stripe con clave restringida `rk_`/`sk_`, PayPal o WooCommerce; credenciales cifradas) · lista sin claves. |
| POST | `…/connections/:id/verify` | Re-valida credenciales. |
| GET | `…/connections/:id/reconcile` | Suscriptores activos separados en `matched` / `unmatched` contra los usuarios por email. |
| POST | `…/connections/:id/invite` | Invita en bloque a los no registrados (hasta 200 emails; resultado por email, nunca falla en bloque). |
| DELETE | `…/connections/:id` | Desconecta y borra la clave cifrada. |

**Tiers**: `GET/POST/PATCH/DELETE …/tiers/catalog[/:id]` (catálogo) · `GET …/user-tiers?userIds=` (tier efectivo de hasta 500 usuarios) · `PUT …/user-tiers/:userId` (tier manual) · `POST …/user-tiers/sync` (reconcilia con los pagos; publica `payment_connections.user_tier.changed`).

**Espejo de pedidos (WooCommerce)**: `POST …/orders-mirror/sync?days=` · `GET …/orders-mirror/expiring?days=` · `GET/PUT …/orders-mirror/rules` (clasificación por regex → `LIFETIME|SUBSCRIPTION|TIMED|ONE_OFF|INFRA`) · `PUT …/orders-mirror/webhook-secret` · `GET …/orders-mirror/webhook-status`.

**Dashboard de suscripciones**: `POST …/subscriptions-dashboard/sync` · `GET …/subscriptions-dashboard[?filtros]` · `GET …/summary` · `GET …/subscribers/:id/renewal-url` · `POST …/subscribers/:id/renewal-email` · `GET/PUT …/renewal-template` · `GET/PUT …/cancel-portal-url` · `POST …/daily/run-now`.

**Autoservicio del usuario** (Bearer): `GET …/me/subscription` (su suscripción externa, para `/cuenta`) · `POST …/me/billing-portal` (sesión del Customer Portal de Stripe).

**Webhook entrante**: `POST /modules/payment-connections/woo-webhook?tenant=<slug>` — público con firma HMAC-SHA256 (`x-wc-webhook-signature`); responde 200 a pings de verificación y a topics ignorados para que WooCommerce no desactive el webhook.

## Aula virtual — `/modules/zoom-live`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/zoom-live/sessions[/:id]` | Bearer | Lista (filtros curso/lección/estado/rango) · detalle. `joinUrl` y grabación **solo** para inscritos o staff. |
| POST | `/modules/zoom-live/sessions/:id/register` · `unregister` | Bearer | Inscripción (idempotente) · cancelación. |
| POST | `/modules/zoom-live/sessions/:id/join` | Bearer | Sella la entrada (proxy de asistencia) y devuelve el `joinUrl`. |
| POST · PUT · DELETE | `/modules/zoom-live/sessions[/:id]` | staff | Crea (`topic`, `startTime`, `durationMinutes` 15-480, `hostEmail`, `timezone`, curso/lección opcional) · edita · cancela (soft). |
| GET | `/modules/zoom-live/sessions/:id/registrations` | staff | Roster de inscritos. |
| GET · POST · PUT | `/modules/zoom-live/sessions/:id/attendance[…]` | staff | Informe de asistencia · sincroniza contra la API de Zoom · override manual por miembro (`{ present: bool\|null }`). |
| POST | `/modules/zoom-live/test-credentials` | admin | Smoke test de las credenciales S2S (`real` o `stub`). |
| GET | `/modules/zoom-live/webhook-events` | admin | Log de webhooks de Zoom (QA). |
| GET | `/modules/zoom-live/sessions/:id/calendar.ics` · `calendar/google` · `outlook` · `office365` | **Público** | Evento de calendario (nunca incluye el `joinUrl`). |
| POST | `/webhooks/zoom` | Firma HMAC de Zoom | `meeting.started/ended`, `recording.completed` + handshake de validación. Idempotente por `event_id`. |

## IA

**Generación de contenido** — `/modules/ai-content` (formador+):

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/modules/ai-content/generate` | Genera un borrador `SUMMARY\|FLASHCARDS\|QUIZ` de una lección: `{ lessonId, courseId, type }`. |
| GET | `/modules/ai-content/drafts[/:id]` | Lista con filtros · detalle. |
| PATCH | `/modules/ai-content/drafts/:id/content` · `publish` · `reject` | Edita el JSON antes de publicar · publica · rechaza con razón. |

Errores: `AI_CONTENT_LESSON_TEXT_EMPTY` 422 · `AI_CONTENT_DRAFT_NOT_IN_DRAFT` 409 · `AI_CONTENT_PROVIDER_ERROR` **503**.

**Corrección con rúbrica** — `/modules/ai-grader` (formador+):

| Método | Ruta | Qué hace |
|---|---|---|
| GET · PUT · DELETE | `/modules/ai-grader/questions/:questionId/rubric` | Rúbrica por pregunta: `instructions` + 1-8 criterios con peso (la suma debe igualar los puntos de la pregunta). |
| POST | `/modules/ai-grader/attempts/:attemptId/suggest` | Genera sugerencias IA para las respuestas abiertas (`{ force? }` ignora la caché). |
| GET | `/modules/ai-grader/attempts/:attemptId/suggestions` | Sugerencias persistidas (sin llamar al modelo). |
| POST | `/modules/ai-grader/suggestions/:id/apply` | Marca la sugerencia como aplicada (auditoría). La nota real se pone en assessments. |

**Tutor** — `/modules/ai-tutor` y `/admin/ai-tutor`:

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/ai-tutor/courses/:courseId/ask` | Bearer | Pregunta al tutor (RAG): `{ question, conversationId?, lessonId?, topK? }` → respuesta + citas. El staff puede probar cualquier curso sin matrícula ni cuota. |
| POST | `/admin/ai-tutor/courses/:courseId/index` · `reindex-all` | admin | Re-indexa un curso · todos los publicados. |

Errores del tutor: `AI_TUTOR_DAILY_QUESTION_QUOTA` / `AI_TUTOR_TOKEN_QUOTA_EXCEEDED` **429** · `AI_TUTOR_COURSE_NOT_INDEXED` 404 · `AI_TUTOR_COURSE_ACCESS_DENIED` 403 · `AI_PROVIDER_NOT_CONFIGURED` **424** · proveedor caído 502.

El panel de revisión del tutor (respuestas, conocimiento validado, informe mensual) está en [Administración → IA](administracion.md#ia-adminai-admin).
