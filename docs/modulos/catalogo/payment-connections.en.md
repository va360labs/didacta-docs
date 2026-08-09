# mod.payment-connections — Payment connections

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

It connects several payment accounts **in read-only mode** (Stripe with a restricted key, plus PayPal and WooCommerce readers) and reconciles their active subscriptions against Didacta users **by email**. On that foundation it builds:

- The **tier** catalog (a manual level assigned by the admin and/or derived from payment), which other modules consume (drip, access groups).
- A **subscription control dashboard** materialised by a sync worker, with renewal links and reminder emails.
- The **order mirror** for external stores (WooCommerce, with a signed webhook) and access classification rules (`LIFETIME`, `SUBSCRIPTION`, `TIMED`…), with expiry notices.
- **Bulk invitations** for subscribers who are not in Didacta yet.

## What it is NOT

It creates no charges: to sell, use [mod.billing](billing.md) and [mod.subscriptions](subscriptions.md). Cancelling is a deep link into the Stripe portal — the model is 100% read-only over the connected accounts.

## How it works

- Matching is by **normalised email** (lowercase + trim) on both sides; `active`, `trialing` and `past_due` all count as active.
- The effective tier resolves as: manual → derived from payment → "Unknown". Changes publish `payment_connections.user_tier.changed`, which `mod.access-groups` consumes to reconcile memberships.
- API keys are stored **encrypted** per connection; they never come back in a GET.
- The dashboard reads only the materialised table — it never queries the providers live.

## Dependencies

None.

## Data model

`mod_payment_connections_connection` · `_log` · `_tier` · `_user_tier` · `_subscriber` (materialised snapshot) · `_order` (order mirror with entitlement and expiry) · `_sync_history`.

## API

Prefix `/modules/payment-connections` (super_admin, except the self-service `me/*` endpoints). Details in [Reference → Payments](../../api/referencia/pagos-directo-ia.md#payment-connections-modulespayment-connections-super_admin).

## Events

**Emits**: `payment_connections.user_tier.changed`. It consumes none.

## Configuration

No variables of its own: credentials, the Woo webhook secret, the renewal template and the cancellation portal URL are all **per tenant**, encrypted in the database. Minimum scopes for the restricted Stripe key: `Customers = Read` + `Subscriptions = Read` (optionally `Invoices = Read`). Sync cron: `PAYMENT_SUBSCRIBERS_SYNC_CRON`.
