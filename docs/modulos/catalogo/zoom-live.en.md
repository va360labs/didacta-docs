# mod.zoom-live — Virtual classroom (Zoom)

<span class="didacta-chip didacta-chip--community">Community</span> · **Live** category (can be disabled)

## What it does

Live classes over Zoom's **Server-to-Server OAuth** API. The instructor schedules sessions (optionally linked to a course or to a specific lesson, which enables the "Join" button in the lesson); students register from a shareable link, and both the `joinUrl` and the **recording** are gated server-side to registered attendees or staff. It includes:

- **Add to calendar** on registering: Google, Outlook.com, Microsoft 365 and `.ics` (the links never contain the `joinUrl`).
- An email + in-app **reminder** 2 hours beforehand (exactly one per class, with an atomic claim to prevent duplicates).
- **Real attendance**: an entry proxy (the click on "Join") reconciled afterwards against Zoom's Report API, with a manual override by the instructor.
- The recording is persisted when the `recording.completed` webhook arrives.

## How it works

- With no credentials configured, the module uses a **deterministic stub** — the whole flow can be tested without a Zoom account.
- Webhooks are processed with an **idempotent log** keyed by `event_id` (Zoom retries up to 3 times) and a verified HMAC signature; this includes Zoom's endpoint validation handshake.
- Zoom participants who match no user (guests, a different email address) are kept with their Zoom identity for manual reconciliation.
- The typical attendance sync error is a missing `report:read:admin` scope on the Zoom app; it is recorded on the session so the instructor knows why there is no data.
- When the class ends, `mod.surveys` can fire the post-class survey automatically.

## Dependencies

Optional: `mod.courses` (linking a session ↔ course/lesson).

## Data model

`mod_zoom_session` (the class: schedule, host, URLs, recording, sync flags) · `_session_registration` (registration, the basis for gating) · `_session_attendance` (entry click + Zoom data + override) · `_webhook_event` (idempotent log).

## API

Prefix `/modules/zoom-live` (+ the public calendar + the webhook at `/webhooks/zoom`). Details in [Reference → Payments, classroom and AI](../../api/referencia/pagos-directo-ia.md#virtual-classroom-moduleszoom-live).

## Events

**Emits**: `zoom.session.created/updated/cancelled/started/ended`, `zoom.session.recording_ready`, `zoom.session.registration.created/cancelled`, `zoom.session.attendance_synced`. It consumes none.

## Configuration

- **Per tenant** (panel): Zoom S2S credentials (`accountId`, `clientId`, `clientSecret`), encrypted at rest.
- **Environment**: `ZOOM_WEBHOOK_SECRET`, `ZOOM_REMINDER_CRON`, `ZOOM_REMINDER_HOURS_BEFORE`, `ZOOM_ATTENDANCE_SYNC_CRON`. See [Configuring Zoom](../../configuracion/zoom.md).
