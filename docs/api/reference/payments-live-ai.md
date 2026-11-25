# Referencia — Pagos, aula virtual e IA

Billing, suscripciones, conexiones de pago, cobros de plataforma (Cloud), Zoom y los tres módulos de IA. Todas las rutas cuelgan de `/api/v1`.

## Pago único (billing) — `/modules/billing`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/billing/offer/:courseId` | Bearer | Oferta del curso (`forSale`, precio, descuento). Tolerante: sin Stripe configurado devuelve `forSale: false`, nunca 500. |
| POST | `/modules/billing/checkout/:courseId` | Bearer | Crea la Checkout Session de Stripe y devuelve la URL. 409 si el curso no está publicado o ya tienes acceso. |
| GET · POST · PATCH · DELETE | `/modules/billing/products[/:id]` | admin | Productos (curso ↔ precio de Stripe): alta con `stripePriceId` existente **o** `amountCents` (crea Product+Price), activación, borrado (no toca órdenes históricas). |
| GET | `/modules/billing/public/catalog` · `public/offer/:courseId` | Público (tenant por dominio) | Catálogo público de venta · oferta pública. Degradan a vacío sin Stripe. La oferta **no filtra existencia**: curso inexistente o sin publicar responde 200 `{ forSale: false, options: [] }`, nunca 404. |
| POST | `/modules/billing/public/checkout/:courseId` | Público | **Checkout anónimo**: el visitante compra sin cuenta. Body `{ optionId?, email? }` — el `email` solo **precarga** el formulario de Stripe; la cuenta se crea SIEMPRE con el email confirmado en el pago. 404 curso inexistente (o `:courseId` no UUID) · 409 no publicado · 503 sin pasarela. |
| GET | `/modules/billing/public/checkout-session/:sessionId` | Público | Vuelta del pago (pantalla de gracias): si la sesión de **esta** academia está pagada, qué curso se compró, cuánto y cuándo. |
| POST | `/modules/billing/webhook` | Firma Stripe (`stripe-signature`) | Recibe `checkout.session.completed/expired`, `charge.refunded`. Idempotente por `stripe_event_id`. |

## Suscripciones — `/modules/subscriptions`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/modules/subscriptions/checkout/:courseId` | Bearer | Checkout `mode=subscription`: `{ stripePriceId }` (recurring). |
| GET | `/modules/subscriptions/me` | Bearer | Mis suscripciones (incluye canceladas, con plan y curso; en la membresía, `tierId` y `tierName` del nivel). Devuelve `[]` si el módulo no está configurado. |
| GET | `/modules/subscriptions/me/:id/invoices` | Bearer (propietario) | Facturas con URL hosted de Stripe. |
| POST | `/modules/subscriptions/me/:id/cancel` | Bearer (propietario) | Cancela: `{ immediate? }` (por defecto al final del periodo). |
| POST | `/modules/subscriptions/me/portal` | Bearer | Abre el portal de cliente de Stripe del alumno (cambiar la tarjeta, ver sus facturas, darse de baja). |
| GET · POST | `/modules/subscriptions/me/:id/leave` | Bearer (propietario) | **Al darse de baja**: GET devuelve los motivos y la pausa o el descuento que el aula ofrece ahora a esa suscripción · POST registra lo que decide: `{ reason, reasonText?, choice: pause \| discount \| cancel, pauseMonths? (1–12) }` —pausar (sin cobro ni acceso hasta el primer cobro tras la pausa), quedarse con el descuento o la baja al final del periodo pagado—. Se confirma por correo. La oferta se configura en `/membership/admin/retention`. |
| POST | `/modules/subscriptions/me/membership/pay-now` | Bearer | Termina el trial de la membresía y cobra ya. |
| GET | `/modules/subscriptions/me/membership/change-options` | Bearer | **Cambiar de nivel**: la membresía viva, a qué precios puede pasar (niveles de pago encendidos, misma moneda; cada uno con `direction`: `upgrade` \| `downgrade` según el precio al mes), el nivel gratis si lo hay y si algo lo impide ahora (`blockedReason`: `no_membership` \| `not_active` \| `cancel_scheduled` \| `pause_scheduled`). Sin niveles, sin opciones. |
| POST | `/modules/subscriptions/me/membership/change-preview` | Bearer | `{ planId }` → lo que pasaría al cambiar ahora: `amountDueNow` (subir), `creditCents` (bajar: a favor en la próxima factura) o `trialing: true` (sin prorrateo). No cambia nada. |
| POST | `/modules/subscriptions/me/membership/change` | Bearer | `{ planId }` → cambia al momento con prorrateo de Stripe: subir cobra la diferencia ya (si falla, 402 y no cambia nada); bajar la deja a favor en la próxima factura; en prueba, sin cobro. Devuelve `{ subscription, direction, charged }`. Ver [Suscripciones → Niveles](../../modules/catalog/subscriptions.md#cambiar-de-nivel-desde-la-cuenta). |
| POST | `/modules/subscriptions/webhook` | Firma Stripe (secret propio) | `customer.subscription.*`, `invoice.*`; fulfillment de membresía idempotente. |
| POST | `/modules/subscriptions/admin/grace-expiration/run-now` | super_admin | Fuerza la expiración de periodos de gracia (QA) → 202. |

Errores comunes: `SUBSCRIPTIONS_PRICE_NOT_RECURRING` 422 · `SUBSCRIPTIONS_ALREADY_ACTIVE` 409 · `*_STRIPE_CONFIG_MISSING` 503 · `*_STRIPE_API_ERROR` 502. Cambio de nivel: `MEMBERSHIP_PLAN_CHANGE_NOT_ALLOWED` 409 · `MEMBERSHIP_PLAN_CHANGE_PAYMENT_FAILED` 402.

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

**Autoservicio del usuario** (Bearer): `GET …/me/subscription` (su suscripción externa, para `/account`) · `POST …/me/billing-portal` (sesión del Customer Portal de Stripe).

**Webhook entrante**: `POST /modules/payment-connections/woo-webhook?tenant=<slug>` — público con firma HMAC-SHA256 (`x-wc-webhook-signature`); responde 200 a pings de verificación y a topics ignorados para que WooCommerce no desactive el webhook.

## Cobros de plataforma — `/modules/payouts` (Cloud)

El **modo plataforma**: el operador cobra las ventas de todas las aulas con su cuenta de Stripe y liquida a cada creador. Solo existe cuando el operador define `PAYMENTS_PLATFORM_STRIPE_SECRET_KEY`; sin ella **no hay ninguna de estas rutas** (404) y el aula cobra con sus propias claves (`/admin/tenant-settings/stripe`). Las rutas que usa el plano de control —volumen, comisión, países— están en [Instancia y plano de control](instance-operations.md#cobros-de-plataforma-rutas-del-pool-cloud). Todo lo del creador exige `tenant_admin` o `super_admin` (`PAYOUTS_ADMIN_ROLE_REQUIRED`).

**Alta de cobros**

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/payouts/status` | Estado de los cobros del aula: `mode` (`platform`, `own` o `none`), si ya puede vender (`ready`) y por qué no (`reason`), la cuenta conectada, el nombre del operador y los países admitidos. Con `?sync=1` consulta antes la cuenta en Stripe (vuelta del alta). |
| POST | `/modules/payouts/connect` | Conecta cobros: `{ country (ISO alfa-2), entityType: individual \| company }`. Crea (o reutiliza) la cuenta conectada y devuelve el enlace del alta alojada de Stripe. Un país no admitido da 422. |
| POST | `/modules/payouts/connect/link` | Enlace nuevo del alta alojada («Completar», o el anterior caducó). |
| GET | `/modules/payouts/dashboard-link` | Enlace de un solo uso al panel Express del creador en Stripe. |
| POST | `/modules/payouts/disconnect` | Desconecta cobros: el aula **deja de vender** (checkout cerrado) y sus suscripciones vivas se cancelan al final del periodo. La cuenta de Stripe **no se cierra** (lo pendiente se sigue pagando); conectar de nuevo la reactiva. |

**Ventas, reembolsos y datos fiscales**

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/payouts/sales?page=&pageSize=` | Ventas del aula cobradas por la plataforma, más recientes primero (`pageSize` máx. 100). |
| GET | `/modules/payouts/summary` | Resumen **por divisa**: vendido, comisiones de tarjeta, comisión del operador, recibido, pendiente y deuda. |
| GET | `/modules/payouts/sales/:id/refund-preview?amount=` | Qué pasaría al reembolsar (`amount` en la moneda del cargo; sin él, todo lo que queda): lo que recibe el alumno y lo que se descuenta al creador. No mueve dinero. |
| POST | `/modules/payouts/sales/:id/refund` | Reembolsa una venta: `{ amount?, requestId? }` (total si no hay `amount`). Devuelve el dinero al alumno, recupera del creador su parte y deja como deuda lo que no se pueda recuperar. `requestId` (UUID) hace el reintento idempotente: un doble clic acaba en **un** reembolso. Venta de otra aula: 404. |
| GET · PUT | `/modules/payouts/tax-profile` | Datos fiscales del creador (sin el NIF entero; si aún no hay, propone lo que ya sabe Stripe) · guárdalos: `{ entityType, legalName, country, taxId?, vatId?, address, birthDate? }`. Valida el NIF (español, con su control) y el NIF-IVA en VIES. |
| GET | `/modules/payouts/invoices` · `/invoices/:id/pdf` | Facturas de la comisión emitidas al aula, más recientes primero · el PDF tal como se emitió (de otra aula: 404). |

**Operador (super_admin)**

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/modules/payouts/admin/invoices/run` | Emite ahora las facturas de un mes (`{ month?: "YYYY-MM", tenantId? }`; por defecto, el anterior) a todas las aulas o a una. Idempotente. |
| POST | `/modules/payouts/admin/reconcile/run-now` | Lanza una pasada del conciliador de cobros (todas las aulas): `{ deep?, windowHours? (1–720) }`. Nunca desde una sesión de soporte. |
| POST | `/modules/payouts/webhook` | Webhook de Stripe de la plataforma. Público, con **firma de Stripe** (secreto del operador); el aula sale del registro de objetos del evento, no de la petición. |

**Errores de dominio** (`code`, todos con prefijo `PAYOUTS_`): `COUNTRY_UNSUPPORTED`, `TAX_PROFILE_INVALID` y `PRICE_CURRENCY` → 422 · `ACCOUNT_NOT_FOUND`, `SALE_NOT_FOUND` e `INVOICE_NOT_FOUND` → 404 · `ACCOUNT_ALREADY_STARTED`, `ACCOUNT_ONBOARDING_INCOMPLETE`, `NOT_READY`, `SALE_NOT_REFUNDABLE`, `REFUND_EXCEEDS_REMAINING` y `REFUND_REJECTED` → 409 · `STRIPE_CONFIG` → 503 · `STRIPE_API_ERROR` → 502 · `WEBHOOK_SIGNATURE_INVALID` → 401.

## Aula virtual — `/modules/zoom-live`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/zoom-live/sessions[/:id]` | Bearer | Lista (filtros curso/lección/estado/rango) · detalle. `joinUrl` y grabación **solo** para inscritos o staff. Una clase de un nivel que quien mira no tiene llega con `locked: true` y `accessLevels` (título, hora y duración; sin descripción, enlace ni grabación). |
| POST | `/modules/zoom-live/sessions/:id/register` · `unregister` | Bearer | Inscripción (idempotente) · cancelación. Clase de un nivel que no tiene: 403 `ZOOM_SESSION_LOCKED`. |
| POST | `/modules/zoom-live/sessions/:id/join` | Bearer | Sella la entrada (proxy de asistencia) y devuelve el `joinUrl`. Clase de un nivel que no tiene: 403 `ZOOM_SESSION_LOCKED`, aunque se hubiera inscrito antes. |
| POST · PUT · DELETE | `/modules/zoom-live/sessions[/:id]` | staff | Crea (`topic`, `startTime`, `durationMinutes` 15-480, `hostEmail`, `timezone`, curso/lección opcional y `accessGroupIds?`: solo para esos niveles; vacío = todos) · edita · cancela (soft). Un grupo que no existe: 422 `ZOOM_ACCESS_GROUP_UNKNOWN`. |
| GET | `/modules/zoom-live/sessions/:id/registrations` | staff | Roster de inscritos. |
| GET · POST · PUT | `/modules/zoom-live/sessions/:id/attendance[…]` | staff | Informe de asistencia · sincroniza contra la API de Zoom · override manual por miembro (`{ present: bool\|null }`). |
| POST | `/modules/zoom-live/test-credentials` | admin | Smoke test de las credenciales S2S (`real` o `stub`). |
| POST | `/modules/zoom-live/test-webhook` | admin | Autocomprobación del webhook de Zoom: Didacta se llama a sí misma por su URL pública con el handshake `endpoint.url_validation` firmado con el Secret Token del aula. Siempre 200 con `{ ok, url, reason?, status?, detail? }`: el motivo del fallo es el dato. |
| POST | `/modules/zoom-live/admin/reminders/run` | formador o admin | Fuerza el barrido de recordatorios de clase sin esperar al cron, acotado al aula de quien llama. |
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
| GET | `/modules/ai-tutor/status` | Bearer | `{ available, reason }`: si el tutor puede contestar en este aula. Sin proveedor de IA (ni propio ni por defecto) o sin el módulo, `available: false` con `reason` `NO_PROVIDER` o `MODULE_INACTIVE`, y la web no lo enseña. |
| POST | `/modules/ai-tutor/courses/:courseId/ask` | Bearer | Pregunta al tutor (RAG): `{ question, conversationId?, lessonId?, topK? }` → respuesta + citas. El staff puede probar cualquier curso sin matrícula ni cuota. |
| POST | `/admin/ai-tutor/courses/:courseId/index` · `reindex-all` | admin | Re-indexa un curso · todos los publicados. |

Errores del tutor: `AI_TUTOR_DAILY_QUESTION_QUOTA` / `AI_TUTOR_TOKEN_QUOTA_EXCEEDED` **429** · `AI_TUTOR_COURSE_NOT_INDEXED` 404 · `AI_TUTOR_COURSE_ACCESS_DENIED` 403 · `AI_PROVIDER_NOT_CONFIGURED` **424** · proveedor caído 502.

El panel de revisión del tutor (respuestas, conocimiento validado, informe mensual) está en [Administración → Proveedores de IA](administration.md#ia-adminai-admin).
