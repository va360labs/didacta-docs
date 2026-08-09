# mod.member-registration — Member registration

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Member sign-up with **manual validation**: request → evidence → human decision. The public wizard (`/inscripcion-miembros`) assembles its steps according to the tenant's policy:

- Verification through **Telegram** (Login Widget + a group membership check).
- **Email OTP** (a 6-digit code).
- Both, or **open registration** (the only gate being manual approval).

Registration creates the user as `PENDING`, kicks off a background **lookup of subscriptions and purchases** for that email across the connected payment accounts, and notifies the **approver** by email with two signed single-use links (approve / reject) plus the overdue payment status if there is one.

## How it works

- The steps are chained with **HMAC-signed tickets** (no state in the database): the Telegram ticket authorises the OTP and the OTP's `verificationToken` authorises registration.
- Approving activates the user, assigns them the **default access group** and sends the welcome email; rejecting deactivates them and sends a notice.
- If the tenant requires a verifier that is not operational (for example Telegram with no bot configured), registration returns a **fail-closed 503** — it never lets anyone through unverified.
- A daily **GDPR purge** worker anonymises the Telegram data of requests that were never decided once the retention window expires (90 days by default).
- The admin panel also supports **manual sign-up** without OTP, re-running the payment lookup, deciding without the email, sending renewal reminders and managing **overdue payment flags** (with CSV import).

## Dependencies

Hard: `mod.payment-connections` (lookup and renewal links) and `mod.access-groups` (default group on approval).

## Data model

`mod_member_registration_profile` (per-user flow data) · `mod_member_registration_decision_token` (single-use tokens, SHA-256 hash, 7-day TTL) · `mod_member_registration_payment_flag` (reference overdue payments) · `mod_member_registration_subscription_lookup` (the lookup result).

## API

Prefix `/modules/member-registration` (public, admin and payment-flags blocks). Details in [Reference → Community and people](../../api/referencia/comunidad.md#member-registration-modulesmember-registration).

## Events

**Emits**: `member_registration.request.created/approved/rejected`. It consumes none.

## Configuration

- **Per tenant** (Administration → Settings → Registration): the Telegram bot (token encrypted at rest), the required verifiers and the approver's email. It registers 4 editable email templates under Administration → Emails.
- **Legacy global fallback** for single-tenant setups: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_GROUP_ID`, `TELEGRAM_BOT_USERNAME`, `MEMBER_APPROVAL_EMAIL`.
- Workers: `MEMBER_PURGE_CRON` (03:00 UTC), `MEMBER_RETENTION_DAYS` (90).
