# mod.subscriptions — Recurring subscriptions

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Recurring monthly/annual subscriptions with Stripe (`mode=subscription`): subscriptions to specific courses and, on top of that, the **membership** behind the public `/unete` page, with its own plans (billing period, price, struck-through price, trial days, featured plan) and its landing configuration (headlines, the access group it grants, the lesson limit during the trial, a testimonial).

## How it works

- Stripe webhooks (`customer.subscription.*`, `invoice.*`) update the local state: `PENDING → ACTIVE → PAST_DUE → UNPAID / CANCELED`.
- A configurable **grace period** after a failed payment (3 days by default): if Stripe charges successfully before it expires, the subscription returns to `ACTIVE`; if it expires, it moves to `UNPAID` and **pauses the enrollment** (an hourly cron expires elapsed grace periods). Dunning is handled by Stripe's automatic retries.
- Cancellation can be at the end of the period (the default) or immediate (unenrolling right away).
- Host bridges connect it to the rest: activation → enrollment; non-payment → pause; membership activated/revoked → granting/removing the access group.
- The membership checkout is **anonymous** (the visitor pays at `/unete` and the account is materialised with the email confirmed by Stripe) and accepts referral codes, which `mod.referrals` interprets.
- It coexists with `mod.billing` in the same tenant (one-off and subscription courses) without sharing tables.

## Dependencies

Hard: `mod.learning`, `mod.courses`.

## Data model

`mod_subscriptions_subscription` (one live subscription per user and course) · `mod_subscriptions_plan` (membership plans + Stripe IDs) · `mod_subscriptions_membership_config` (the copy and rules of `/unete`) · `mod_subscriptions_invoice` · `mod_subscriptions_webhook_event` (idempotent log).

## API

Prefixes `/modules/subscriptions` (student + webhook) and `/membership` (public + admin). Details in [Reference → Payments](../../api/referencia/pagos-directo-ia.md#subscriptions-modulessubscriptions) and [Reference → Community](../../api/referencia/comunidad.md#membership-membership).

## Events

**Emits**: `subscriptions.subscription.created/activated/past_due/unpaid/canceled`, `subscriptions.invoice.paid/payment_failed/refunded`, `subscriptions.membership.activated/revoked`. It consumes none.

## Configuration

Stripe (secret key + webhook) is configured per tenant under Settings → Payments, shared with `mod.billing`. With no credentials (neither tenant nor instance) checkout returns 503 — plans, trials and the access group remain editable regardless, as they do not depend on Stripe.

| Variable | What it is for |
| --- | --- |
| `STRIPE_SECRET_KEY` | Instance fallback: only if the tenant has not configured its key in the panel. |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | A dedicated webhook (keeping it separate from billing's is recommended); instance fallback. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | Grace period after a failed payment (3). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` / `CANCEL_URL_BASE` | Checkout return URLs. |
| `SUBSCRIPTIONS_DAILY_CRON` / `DAILY_TZ` / `RENEWAL_WINDOW_DAYS` | Daily summary and renewal notices. |

Plans, trials and the access group: per tenant, from the panel.

Out of scope for the MVP: plan changes with proration, B2B invoicing and pause/resume — cancellation is the only way out.
