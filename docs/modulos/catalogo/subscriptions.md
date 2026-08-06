# mod.subscriptions — Suscripciones recurrentes

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Suscripciones recurrentes mensuales/anuales con Stripe (`mode=subscription`): suscripción a cursos concretos y, sobre esa base, la **membresía** de la página pública `/unete`, con planes propios (periodicidad, precio, precio tachado, días de trial, plan destacado) y su configuración de landing (titulares, grupo de acceso que concede, límite de lecciones en trial, testimonial).

## Cómo funciona

- Los webhooks de Stripe (`customer.subscription.*`, `invoice.*`) actualizan el estado local: `PENDING → ACTIVE → PAST_DUE → UNPAID / CANCELED`.
- **Periodo de gracia** configurable tras un impago (3 días por defecto): si Stripe cobra antes de expirar, vuelve a `ACTIVE`; si expira, pasa a `UNPAID` y **pausa la matrícula** (un cron horario expira las gracias vencidas). El dunning lo hacen los reintentos automáticos de Stripe.
- La cancelación puede ser al final del periodo (default) o inmediata (desenrola al momento).
- Bridges del host conectan con el resto: activación → matrícula; impago → pausa; membresía activada/revocada → concesión/retirada del grupo de acceso.
- El checkout de membresía es **anónimo** (el visitante paga en `/unete` y la cuenta se materializa con el email confirmado por Stripe) y acepta códigos de referido, que interpreta `mod.referrals`.
- Convive con `mod.billing` en el mismo tenant (cursos one-shot y por suscripción) sin compartir tablas.

## Dependencias

Duras: `mod.learning`, `mod.courses`.

## Modelo de datos

`mod_subscriptions_subscription` (una viva por usuario y curso) · `mod_subscriptions_plan` (planes de membresía + IDs de Stripe) · `mod_subscriptions_membership_config` (copy y reglas de `/unete`) · `mod_subscriptions_invoice` · `mod_subscriptions_webhook_event` (log idempotente).

## API

Prefijos `/modules/subscriptions` (alumno + webhook) y `/membership` (público + admin). Detalle en [Referencia → Pagos](../../api/referencia/pagos-directo-ia.md#suscripciones-modulessubscriptions) y [Referencia → Comunidad](../../api/referencia/comunidad.md#membresia-membership).

## Eventos

**Emite**: `subscriptions.subscription.created/activated/past_due/unpaid/canceled`, `subscriptions.invoice.paid/payment_failed/refunded`, `subscriptions.membership.activated/revoked`. No consume.

## Configuración

Stripe (clave secreta + webhook) se configura por tenant en Administración → Pagos, compartida con `mod.billing`. Sin credenciales (ni de tenant ni de instancia) el checkout responde 503 — planes, trial y grupo de acceso siguen editables igualmente, no dependen de Stripe.

| Variable | Para qué |
| --- | --- |
| `STRIPE_SECRET_KEY` | Fallback de instancia: solo si el tenant no configuró su clave en el panel. |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | Webhook dedicado (recomendado separarlo del de billing); fallback de instancia. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | Gracia tras impago (3). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` / `CANCEL_URL_BASE` | Retorno del checkout. |
| `SUBSCRIPTIONS_DAILY_CRON` / `DAILY_TZ` / `RENEWAL_WINDOW_DAYS` | Resumen diario y avisos de renovación. |

Planes, trial y grupo de acceso: por tenant, desde el panel.

Fuera del MVP: cambio de plan con prorrateo, facturación B2B y pause/resume — la cancelación es la única salida.
