# mod.referrals — Referral program

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

The membership referral program: every member has their own code and link (`/unete?ref=CODE`) and, if someone arrives through it and **pays**, the referrer earns a percentage of the actual charge. The operator configures everything from the panel without touching code: percentage (`commissionBps`), scope (first payment only, or recurring with a month limit), attribution window, guarantee period, minimum payout and whether the referrer must hold an active membership.

## How it works

The full cycle: click (deduplicated by code + day + **hash of the IP address** — the IP is never stored in the clear) → checkout with the code in Stripe's metadata → **attribution** during webhook fulfillment (idempotent and anti-self-referral) → a `PENDING` commission for every invoice paid within scope → `APPROVED` once the guarantee period elapses (automatic worker) → **manual payout** by the admin with an external reference → `PAID`.

- A refund **automatically revokes** a commission in `PENDING`/`APPROVED`; one already `PAID` is left alone (the operator handles the clawback manually).
- The referrer gets an in-app notification + email when they earn a commission and when a payout is settled (templates editable under Administration → Emails).
- With the program disabled, the public endpoints behave as if it did not exist (404).

## Dependencies

Optional: `mod.subscriptions` (checking that the referrer's membership is live).

## Data model

`mod_referrals_config` (per-tenant policy) · `mod_referrals_code` (one unique code per user) · `mod_referrals_click` (deduplicated clicks) · `mod_referrals_referral` (attribution) · `mod_referrals_commission` (one per Stripe invoice) · `mod_referrals_payout` (payouts).

## API

Prefix `/modules/referrals` (member + public `track` + admin). Details in [Reference → Community and people](../../api/referencia/comunidad.md#referrals-modulesreferrals).

## Events

- **Emits**: `referrals.referral.attributed`, `referrals.commission.created/approved/revoked`, `referrals.payout.recorded`.
- **Consumes** (through a bridge): `subscriptions.membership.activated`, `subscriptions.invoice.paid`, `subscriptions.invoice.refunded`.

## Configuration

Everything is per tenant, under Administration → Referrals. No environment variables of its own (approval cron: `REFERRALS_APPROVAL_CRON`). v1 without Stripe Connect: payouts are recorded by hand with an external reference.
