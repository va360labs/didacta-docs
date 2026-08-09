# mod.billing — Stripe Checkout (one-off payment)

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Course monetization with one-off payments through **Stripe Checkout**: the admin links each course to a Stripe price (or creates one from Didacta), the student clicks buy, the backend creates the Checkout Session and redirects to the hosted checkout. Once the payment is confirmed, the webhook completes the order and emits `billing.order.completed`, which `mod.learning` listens to in order to enroll the student with origin `PURCHASE`.

It also covers the **public purchase journey**: a visitor **with no account** buys from the public catalog (`/catalogo`) or from a course's sales page (`/catalogo/<slug>`); the order starts as `PENDING` with no owner and fulfillment materialises the account with the email confirmed by Stripe (find-or-create + a welcome email with a "set your password" link, template `billing.welcome`, editable per tenant under Administration → Emails). The return pages for a public payment are `/catalogo/checkout/success` and `/catalogo/checkout/cancel`.

## How it works

- **Explicit idempotency**: every Stripe `evt_*` event is persisted and never reprocessed — redelivering an anonymous checkout duplicates neither the account nor the enrollment.
- Guards before charging: 404 if the course does not exist, 409 if it is not published, 409 if you already have access.
- With no Stripe configured the module does not start up: the public catalog returns an empty list and checkout returns 503 with a clear message. The rest of the platform is unaffected.
- Refunds are issued from the Stripe dashboard; the `charge.refunded` webhook is processed and flags the order.

## Dependencies

Hard: `mod.learning`, `mod.courses`.

## Data model

`mod_billing_product` (sellable course ↔ price, unique per course and per price) · `mod_billing_order` (the purchase: `PENDING → COMPLETED | CANCELLED | FAILED | REFUNDED`, with a nullable `user_id` for anonymous purchases) · `mod_billing_webhook_event` (idempotent log, PK = `stripe_event_id`).

## API

Prefix `/modules/billing`: authenticated checkout, public surface (`public/catalog`, `public/offer`, `public/checkout`), admin CRUD for products, and the webhook. Details in [Reference → Payments](../../api/referencia/pagos-directo-ia.md#one-off-payment-billing-modulesbilling).

## Events

**Emits**: `billing.order.created/completed/failed/refunded`. It consumes none.

## Configuration

Stripe is configured **per tenant** under Administration → Settings → Payments (encrypted credentials), shared with `mod.subscriptions` — a single key pair per academy. `mod.billing` always registers itself; with no credentials (neither tenant nor instance) checkout returns 503 and the rest of the app stays operational.

| Variable | What it is for |
| --- | --- |
| `STRIPE_SECRET_KEY` / `STRIPE_WEBHOOK_SECRET` | Instance fallback: used only if the tenant has not configured its own in the panel. |
| `BILLING_SUCCESS_URL_BASE` / `BILLING_CANCEL_URL_BASE` | Return URLs for the **authenticated** checkout (default: `/cursos` on the instance domain). The **public** checkout ignores them: it always returns to `/catalogo/checkout/success` and `/catalogo/checkout/cancel` on the tenant's domain. |
