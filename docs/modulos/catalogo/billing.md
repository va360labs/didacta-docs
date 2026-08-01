# mod.billing — Stripe Checkout (pago único)

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Monetización de cursos con pago único vía **Stripe Checkout**: el admin liga cada curso a un precio de Stripe (o lo crea desde Didacta), el alumno pulsa comprar, el backend crea la Checkout Session y redirige al checkout hosted. Al confirmarse el pago, el webhook completa la orden y emite `billing.order.completed`, que `mod.learning` escucha para matricular con origen `PURCHASE`.

Incluye además el **viaje de compra pública**: un visitante **sin cuenta** compra desde el catálogo público (`/catalogo`); la orden nace `PENDING` sin dueño y el fulfillment materializa la cuenta con el email confirmado por Stripe (find-or-create + email de bienvenida con enlace «Define tu contraseña»).

## Cómo funciona

- **Idempotencia explícita**: cada evento `evt_*` de Stripe se persiste y no se reprocesa — la reentrega de un checkout anónimo no duplica cuenta ni matrícula.
- Guardas previas al cobro: 404 curso inexistente, 409 no publicado, 409 si ya tienes acceso.
- Sin Stripe configurado, el módulo no arranca: el catálogo público devuelve lista vacía y el checkout responde 503 con mensaje claro. El resto de la plataforma no se ve afectado.
- Los reembolsos se lanzan desde el dashboard de Stripe; el webhook `charge.refunded` sí se procesa y marca la orden.

## Dependencias

Duras: `mod.learning`, `mod.courses`.

## Modelo de datos

`mod_billing_product` (curso vendible ↔ precio, unicidad por curso y por precio) · `mod_billing_order` (compra: `PENDING → COMPLETED | CANCELLED | FAILED | REFUNDED`, `user_id` nullable para compra anónima) · `mod_billing_webhook_event` (log idempotente, PK = `stripe_event_id`).

## API

Prefijo `/modules/billing`: checkout autenticado, superficie pública (`public/catalog`, `public/offer`, `public/checkout`), CRUD admin de productos y webhook. Detalle en [Referencia → Pagos](../../api/referencia/pagos-directo-ia.md#pago-unico-billing-modulesbilling).

## Eventos

**Emite**: `billing.order.created/completed/failed/refunded`. No consume.

## Configuración

| Variable | Para qué |
| --- | --- |
| `STRIPE_SECRET_KEY` | Sin ella el módulo no arranca (compartida con `mod.subscriptions`). |
| `STRIPE_WEBHOOK_SECRET` | Firma del webhook de billing. |
| `BILLING_SUCCESS_URL_BASE` / `BILLING_CANCEL_URL_BASE` | Retorno del checkout (defaults sensatos). |
