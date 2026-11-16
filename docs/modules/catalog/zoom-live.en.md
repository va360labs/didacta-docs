# mod.zoom-live — Virtual classroom (Zoom)

<span class="didacta-chip didacta-chip--community">Community</span> · **Live** category (can be disabled)

## What it does

Live classes over Zoom's **Server-to-Server OAuth** API. The instructor schedules sessions (optionally linked to a course or to a specific lesson, which enables the "Join" button in the lesson); students register from a shareable link, and both the `joinUrl` and the **recording** are gated server-side to registered attendees or staff. It includes:

- **Add to calendar** on registering: Google, Outlook.com, Microsoft 365 and `.ics` (the links never contain the `joinUrl`).
- An email + in-app **reminder** 2 hours beforehand (exactly one per class, with an atomic claim to prevent duplicates). If the class is **rescheduled**, the reminder is reopened and sent again 2 hours before the new time.
- **Real attendance**: an entry proxy (the click on "Join") reconciled afterwards against Zoom's Report API, with a manual override by the instructor.
- The recording is persisted when the `recording.completed` webhook arrives (Zoom's share link and the duration are stored).
- **Class with your own link**: besides Zoom, a class can carry the link of a third-party video call (Google Meet, Teams…); `https://` only, and that class does not talk to Zoom.

## How it works

- Without Zoom credentials, the classroom does not make up links: in production, creating a session returns "Paste the link to your video call (Zoom, Google Meet or Teams) or connect your Zoom account in Settings." (`ZOOM_NOT_CONFIGURED`). In **simple mode**, the "Live" form accepts the link of your own video call (through the API it is the `externalJoinUrl` field, which can only be changed on classes created with their own link). The development **deterministic stub** (to test the flow without a Zoom account) only works outside production or with `ZOOM_ALLOW_STUB=true`.
- Webhooks are processed with an **idempotent log** keyed by `event_id` (Zoom retries up to 3 times) and a verified HMAC signature; this includes Zoom's endpoint validation handshake.
- **Meetings opened out of hours**: a Zoom notice that happens with the class still far away is not applied. A `meeting.started` more than 30 minutes before the start time, a `meeting.ended` earlier than the start time, or a `recording.completed` for a recording that ended before the class began are archived as ignored (in the notice list they read "Outside the class time: not applied"). So testing the link the day before does not close the class, trigger the post-class survey or attach the test recording. The time that counts is the event's (signed by Zoom), not when the notice arrives.
- Zoom participants who match no user (guests, a different email address) are kept with their Zoom identity for manual reconciliation.
- A Zoom notice signed by one classroom never touches another classroom's classes, and a cancelled class is not reopened by a redelivered `meeting.started`.
- The calendar event (`.ics` and calendar links) carries a stable identifier per class — re-importing it updates the entry instead of duplicating it — and its text comes out in the language of whoever opens it or, if unknown, of the classroom (es, en, pt).
- The typical attendance sync error is a missing `report:read:admin` scope on the Zoom app; it is recorded on the session so the instructor knows why there is no data.
- When the class ends, `mod.surveys` can fire the post-class survey automatically.

## Configuration

- **Enabling or disabling the module**: `/admin/settings?tab=modules` (**Modules** tab) — a per-organization switch.
- **Zoom credentials, per tenant**: `/admin/settings?tab=virtual-classroom`, **Virtual classroom** tab (contributed by the module; it disappears if you disable it). Fields **Account ID**, **Client ID**, **Client Secret** and **Webhook Secret Token**, with **Save credentials**, **Test credentials** and **Delete credentials** buttons. They are stored encrypted (AES-256-GCM) and never returned in plain text; without credentials, see the previous point.
- **Webhook**: the endpoint is `https://your-domain/api/v1/webhooks/zoom`; the **Zoom webhook** block on the same tab shows the exact URL to paste into Zoom (with a **Copy** button). Zoom's **Secret Token** is pasted into the same panel form — it is **no longer** the `ZOOM_WEBHOOK_SECRET` environment variable (that one was global while the S2S credentials were already per tenant). Save it **before** clicking "Validate" in Zoom: the validation is signed with that token and, without it, Didacta answers 401. The **Check webhook** button makes Didacta call itself through its public URL with the same handshake and tells you whether the token is missing, whether the token is not the app's, whether the domain does not resolve to this classroom, or whether something in front (DNS, TLS, proxy) answers in its place.
- **Received notices**: the **Zoom** tab of `/admin/webhooks?tab=zoom` (the "Webhooks" menu entry) lists every notice with its event, meeting and result (OK, ignored or error), with filters by type and result. It is the place to diagnose classes that do not change state or recordings that do not show up (the old `/admin/zoom/webhook-events` route redirects there).
- **Environment (workers)**: `ZOOM_REMINDER_CRON`, `ZOOM_REMINDER_HOURS_BEFORE`, `ZOOM_ATTENDANCE_SYNC_CRON` control reminders and attendance reconciliation (they require Redis). Defaults: reminder check every 5 minutes (`*/5 * * * *`) with 2 hours of notice, and attendance sync every 10 minutes (`*/10 * * * *`). The defaults table and the Zoom App Marketplace setup live in [Configuring Zoom](../../configuration/zoom.md).
- Nothing in this module requires an Enterprise license.

!!! note "Screenshots in full mode"
    The screenshots on this page come from **full mode** (side menu), the mode a new installation starts in. In **simple mode** the same routes are still available from "More", except for the screens that mode replaces with a simplified version (the course editor, Settings, Sales, Students, Live, the community and Referrals); to use them as shown here, switch to full mode under **Settings → "Use the full mode"**.

![The "Virtual classroom" Settings tab with the Zoom Server-to-Server credentials form](../../assets/modules/zoom-live/en/zoom-live-1-credentials.png)

## Step by step

As an instructor (or admin):

1. Open **Virtual classroom** in the instructor menu (`/studio/live`) and click **New session**.
2. Fill in **Title**, **Date and time**, **Duration (min)** (15–480) and **Time zone** — the time is interpreted in the chosen zone, not in your computer's. The **Host email** must have a user in your Zoom account: the meeting is created under their name.
3. Optional: link a **Course (optional)** and a **Lesson (optional)** — with a lesson, enrolled students see the session in its detail. Leave **Announce in the community** checked to publish a feed card people can register from.
4. Click **Create session**, then **Copy link** to share the registration page (`/live/<id>`).
5. **Edit** only allows changing title, date, duration, time zone and agenda; the host, course and lesson are fixed at creation. **Cancel** notifies everyone registered.
6. At class time, click **Start** (or **Start (host)** from the class page).

![The instructor's virtual classroom list with a scheduled class](../../assets/modules/zoom-live/en/zoom-live-2-live-sessions.png)

As a student:

1. Open the `/live/<id>` link (or the class from `/calendar`); with no session you go through sign-in and come back.
2. Click **Register**. The **Add to calendar** dialog opens (Google Calendar, Outlook.com, Outlook (Microsoft 365), Apple Calendar and others via `.ics`).
3. The **Join the class** / **Join now** button only appears if you are registered; the click seals your entry as attendance evidence. You get a reminder 2 hours beforehand.
4. Once the class ends, **Watch recording** appears for registered attendees as soon as Zoom delivers the recording.

![The class page with the registration button and the session status](../../assets/modules/zoom-live/en/zoom-live-3-student-class.png)

Attendance (staff, on the same class page):

1. The **Attendance** panel lists registered members and unmatched Zoom participants.
2. **Sync with Zoom** reconciles against the Report API; each row shows how strong the evidence is.
3. The **Mark** button cycles the manual override: **Present** → **Absent** → back to the automatic calculation.

![The class attendance panel with sync and manual marking](../../assets/modules/zoom-live/en/zoom-live-4-attendance.png)

## Dependencies

Optional: `mod.courses` (linking a session ↔ course/lesson).

## Data model

`mod_zoom_session` (the class: schedule, host, URLs, recording, sync flags) · `_session_registration` (registration, the basis for gating) · `_session_attendance` (entry click + Zoom data + override) · `_webhook_event` (idempotent log).

Manifest permissions: `zoom.session.read` and `zoom.session.write`.

## API

Prefix `/modules/zoom-live` (sessions, registration, `join`, attendance, `test-credentials`, `test-webhook` and `webhook-events`) + the calendar (`sessions/:id/calendar.ics`, `calendar/google`, `calendar/outlook`, `calendar/office365`) + the webhook at `/webhooks/zoom`. Details in [Reference → Payments, classroom and AI](../../api/reference/payments-live-ai.md#virtual-classroom-moduleszoom-live).

## Events

**Emits**: `zoom.session.created/updated/cancelled/started/ended`, `zoom.session.recording_ready`, `zoom.session.registration.created/cancelled`, `zoom.session.attendance_synced`. It consumes none.
