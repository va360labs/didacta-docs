# mod.subscriptions — Recurring subscriptions

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Recurring subscriptions with Stripe (`mode=subscription`), with a configurable billing period of 1 to 12 months (monthly, quarterly, semi-annual, annual…): subscriptions to specific courses and, on top of that, the **membership** behind the public `/join` page, with its own plans (billing period, price, struck-through price, trial days, featured plan) and its landing configuration (headlines, the access group it grants, the lesson limit during the trial, a testimonial).

## How it works

- Stripe webhooks (`customer.subscription.*`, `invoice.*`) update the local state: `PENDING → TRIALING / ACTIVE → PAST_DUE → UNPAID / CANCELED`.
- A configurable **grace period** after a failed payment (3 days by default): if Stripe charges successfully before it expires, the subscription returns to `ACTIVE`; if it expires, it moves to `UNPAID` and **pauses the enrollment** (an hourly cron expires elapsed grace periods). Dunning is handled by Stripe's automatic retries. A trial that never paid gets no grace: if its first charge fails, access is lost immediately.
- Cancellation can be at the end of the period (the default) or immediate (unenrolling right away) through the API; the student, from their account, only has **"Cancel my subscription"**, which opens the **cancellation offer** (see below) and ends in cancellation at the end of the paid period.
- Host bridges connect it to the rest: activation → enrollment; non-payment → pause; membership activated/revoked → granting/removing the access group.
- The membership checkout is **anonymous** (the visitor pays at `/join` and the account is materialised with the email confirmed by Stripe) and accepts referral codes, which `mod.referrals` interprets. A second membership is not sold to someone who already has a live one (409 `MEMBERSHIP_ALREADY_SUBSCRIBED`).
- As with course sales, without the **withdrawal-waiver checkbox** nothing is charged (422), and if the account that collects the money has a **publishable key**, payment happens **inside `/join`** (embedded Checkout; the return is `/join?status=success&session_id=…`, which asks the API whether it was paid and grants nothing by itself). Details in [mod.billing](billing.md).
- With **platform payments** ([mod.payouts](payouts.md)), if the creator's account cannot sell yet, `/join` shows no plans and checkout answers 409 `PAYOUTS_NOT_READY`.
- Every charge produces a receipt email (purchase, renewal, failed payment) that the creator can customise; see [mod.billing → Sales emails](billing.md#sales-emails). The subscription stores the buyer's country and the coupon applied, and shows up on the **Sales** screen ([mod.billing → Sales](billing.md#sales)).
- It coexists with `mod.billing` in the same tenant (one-off and subscription courses) without sharing tables.

!!! note "Screenshots in full mode"
    The screenshots on this page come from **full mode** (side menu), the mode a new installation starts in. In **simple mode** the same routes are still available from "More", except for the screens that mode replaces with a simplified version (the course editor, Settings, Sales, Students, Live, the community and Referrals); to use them as shown here, switch to full mode under **Settings → "Use the full mode"**.

## Configuration

The module belongs to the **core** category: it is active in every tenant and has no switch of its own. Nothing below requires an Enterprise licence.

### Stripe credentials (per tenant)

The same ones as `mod.billing`: under `/admin/settings?tab=payments` (the **Payments** tab), card **"Payments · Stripe"** (including the **"Publishable key (optional)"** to pay inside `/join`). This module adds one field of its own: **"Subscriptions webhook secret (optional)"** — leave it empty to reuse the same secret as the one-off course webhook. With no credentials (neither tenant nor instance) checkout returns 503 — plans, trials and the access group remain editable regardless, as they do not depend on Stripe.

### Webhook to register in Stripe

The card shows the exact URL. For this module (recommended as an endpoint separate from billing's):

- Endpoint: `https://<your-academy-domain>/api/v1/modules/subscriptions/webhook`
- Events to select: `checkout.session.completed`, `checkout.session.expired`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`, `charge.refunded`.

!!! danger "`invoice.paid` and `invoice.payment_failed` hold up the whole lifecycle"
    They are not optional, and they are not "just for the history". If they are missing:

    - **There is no cut-off for non-payment.** `invoice.payment_failed` is what marks the subscription past due, starts the grace period and eventually revokes access. Without it, whoever stops paying **keeps the membership indefinitely** and shows as active in the admin panel.
    - **Trials never convert.** Whoever pays at the end of the trial stays stuck with the trial lesson limit.
    - **The student sees no invoices** in their account, however many receipts Stripe has charged.

    None of this raises an error in any log: an event that never arrives is never missed. If in doubt, check it the other way round — take one payment in test mode and confirm the invoice shows up in the student's account.

### Payment methods

Whichever you have enabled in your Stripe dashboard, same as for single-course sales. Stripe automatically filters out anything that doesn't work for recurring charges, so the list a membership buyer sees may be shorter than the one for a single course.

### Membership plans and page

Everything under `/admin/membership` (**"Membership"**). If Stripe is not configured, the page warns you ("you can prepare plans and the page, but real checkout will fail…").

- **"Plans"** card: fields **"Name"**, **"Billing interval"** (monthly / quarterly / semi-annual / annual / every N months), **"Currency"**, **"Price"**, **"Strikethrough price (optional)"**, **"Trial days"**, toggle **"Preselected on the page"**; button **"Create plan"**. The Stripe Product/Price is created lazily on the first checkout and **rotated** when the amount, currency or period changes (Stripe prices are immutable); renaming the plan renames the Product. A plan with sales is never deleted: it is deactivated. Through the API a plan also accepts fixed per-market prices (`currencyOptions`, currency → amount).
- **"Public page"** card: toggle **"Purchase page active"**, **"Title"**, **"Subtitle"**, **"Access group it grants"**, **"Show the course catalog on the page"**, **"Lessons visible per course during the trial period"** (0 = no limit), per-course reference prices and an optional testimonial; button **"Save configuration"**.

- **"When someone wants to leave"** card (cancellation offer; in simple mode it lives in Settings): see the next section. Further down, **"Who wants to leave and why"** summarises the last days.

![Membership administration: plans and public page configuration](../../assets/modules/subscriptions/en/subscriptions-1-membership-plans.png)

!!! note "The screenshot predates the cancellation cards"
    The screenshot shows the plans and the public page; it does not include the **"When someone wants to leave"** and **"Who wants to leave and why"** cards, which were added later.

### Cancellation offer

When a student clicks **"Cancel my subscription"** under `/account?tab=subscription` (it is the only way to cancel: Stripe's portal no longer offers it), the dialog asks for the **reason** (*It's too expensive*, *I don't have time*, *It wasn't what I expected*, *I learned what I wanted*, *Topics I care about are missing*, *Something else*, with optional free text) and, if the academy has enabled it, offers a way to stay:

- **Offer a pause**: no payment and no access to the content; their account, progress and certificates stay and it restarts on its own. The **"Up to"** setting is 1 to 3 months (3 by default). It skips whole charges, so on a monthly plan any number of months up to the maximum is valid, on a quarterly plan only 3, and on a yearly plan it is not offered. One pause per person every 365 days.
- **Offer a discount**: **"Discount"** from 5 to 90 % (30 by default) **"For"** 1 to 12 months (3 by default). On monthly plans it applies to the next payments; on quarterly or yearly plans, to the next renewal. Once per person.

Both start **switched off**. The card is saved with **"Save offer"**. What the student chooses (pause, discount or cancellation at the end of the period) is confirmed by email, shown on their tab (*Pause scheduled…*, *Paused: restarts on its own on…*, *…% off until…*) and counted in the creator's summary. A sweep (the same hourly worker as the grace periods) starts pauses when their date arrives and emits `subscriptions.subscription.paused`. API: `GET|PUT /membership/admin/retention`, `GET /membership/admin/retention/summary`, and for the student `GET|POST /modules/subscriptions/me/:id/leave`.

### Environment variables

| Variable | What it is for |
| --- | --- |
| `STRIPE_SECRET_KEY` / `STRIPE_PUBLISHABLE_KEY` | Instance fallback: only if the tenant has not configured its keys in the panel. See also `STRIPE_ALLOW_GLOBAL_FALLBACK` in [mod.billing](billing.md). |
| `SUBSCRIPTIONS_WEBHOOK_SECRET` | A dedicated webhook (keeping it separate from billing's is recommended); instance fallback. |
| `SUBSCRIPTIONS_GRACE_PERIOD_DAYS` | Grace period after a failed payment (3). |
| `SUBSCRIPTIONS_GRACE_EXPIRATION_CRON` | Cron expiring elapsed grace periods and starting pauses whose date has arrived (`0 * * * *`, hourly UTC). |
| `SUBSCRIPTIONS_SUCCESS_URL_BASE` / `SUBSCRIPTIONS_CANCEL_URL_BASE` | Return URLs for the per-course subscription checkout (default: `/account/subscriptions` on the instance domain, which redirects to `/account?tab=subscription`). |

The daily summary and renewal notices (`SUBSCRIPTIONS_DAILY_CRON`, `SUBSCRIPTIONS_DAILY_TZ`, `SUBSCRIPTIONS_RENEWAL_WINDOW_DAYS`) belong to [mod.payment-connections](payment-connections.md), which is the module watching external subscriptions.

Out of scope for the MVP: plan changes with proration, B2B invoicing and pause/resume — cancellation is the only way out.

## Step-by-step usage

1. Configure Stripe under `/admin/settings?tab=payments` and register the subscriptions webhook with the URL and events above.
2. Under `/admin/membership`, create the plans: **"Name"**, **"Billing interval"**, **"Price"**, **"Trial days"** if you want a trial, and mark one as **"Preselected on the page"** → **"Create plan"**.
3. In the **"Public page"** card, enable **"Purchase page active"**, pick the **"Access group it grants"** (that is what unlocks the courses upon payment) and, if you use a trial, set **"Lessons visible per course during the trial period"** → **"Save configuration"**.
4. Share `/join`. The visitor picks a plan and clicks **"Continue to secure payment"** ("Payment processed by Stripe · Cancel anytime"; coupons can be applied on the payment screen). On return (`/join?status=success`) they see "Payment received!" and get the email to **set their password**; the account is created automatically and the access group granted through an event.

5. The student manages their subscription under `/account?tab=subscription` (the **"Subscription"** tab): trial and next-charge dates, **"Pay now and activate"** (ends the trial and charges right away), **"Cancel my subscription"** (the cancellation offer, see above) and **"View invoices"** (with a link to the Stripe PDF; if they paid with referral balance, the invoice says so: "Paid with referral balance"). With platform payments there is also **"Manage my subscription"** (Stripe portal to change the card and view invoices).

6. Failed payments: nothing to do on your side. Stripe retries the charge; during the grace period the student keeps access (`PAST_DUE`) and, if the grace elapses unpaid, the enrollment is paused (`UNPAID`) until a later charge reactivates it.
7. The **per-course** subscription (besides the membership) exists today only through the API: `POST /modules/subscriptions/checkout/<courseId>` with a recurring Stripe price — the web app has no button for it.

## Permissions

`subscriptions.subscription.read`, `subscriptions.subscription.create` and `subscriptions.subscription.cancel`.

## Dependencies

Hard: `mod.learning`, `mod.courses`.

## Data model

`mod_subscriptions_subscription` (one live subscription per user and course) · `mod_subscriptions_plan` (membership plans + Stripe IDs) · `mod_subscriptions_membership_config` (the copy and rules of `/join`) · `mod_subscriptions_invoice` · `mod_subscriptions_webhook_event` (idempotent log) · `mod_subscriptions_retention_config` (cancellation-offer settings) · `mod_subscriptions_exit` (each cancellation: reason and what the student chose).

## API

Prefixes `/modules/subscriptions` (student + webhook) and `/membership` (public + admin). Details in [Reference → Payments](../../api/reference/payments-live-ai.md#subscriptions-modulessubscriptions) and [Reference → Community](../../api/reference/community.md#membership-membership).

## Events

**Emits**: `subscriptions.subscription.created/activated/past_due/unpaid/canceled/paused`, `subscriptions.invoice.paid/payment_failed/refunded/partially_refunded`, `subscriptions.membership.activated`, `subscriptions.retention.paused/discounted/canceled`. It consumes none. There is no `membership.revoked`: a membership losing access travels in `subscriptions.subscription.canceled` (immediate cancellation), `.unpaid` and `.paused` with `planId`, which is what the groups bridge listens to.
