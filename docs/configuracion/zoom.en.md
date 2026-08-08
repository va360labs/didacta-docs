# Virtual classroom (Zoom)

Didacta's live classes use **Zoom** (Server-to-Server API + Web SDK) through the `mod.zoom-live` module: sessions, registration with a shareable link, access to the `joinUrl` and the recording gated by enrollment, and attendance reconciled against the Zoom API.

## Credentials: per tenant, not per environment

Zoom's **Server-to-Server OAuth** credentials (Account ID, Client ID, Client Secret) are **not environment variables**: they are configured per organization in the admin panel and stored **encrypted at rest** (hence the importance of [`TENANT_SETTINGS_ENC_KEY`](variables-de-entorno.md#5-authentication-sessions-and-encryption)).

You need a *Server-to-Server OAuth* app created in your account's [Zoom App Marketplace](https://marketplace.zoom.us/), with meeting and recording scopes.

## Zoom webhook

So that Didacta receives events (meeting start/end, recordings available, attendance):

1. In your Zoom app, point the webhook endpoint at your instance:
   ```
   https://your-domain/api/v1/webhooks/zoom
   ```
2. Copy Zoom's **Secret Token** into the variable:
   ```bash
   ZOOM_WEBHOOK_SECRET=...
   ```

Didacta verifies the HMAC-SHA256 signature of every event against the raw request body; without the secret, events are rejected.

## Scheduled jobs

| Variable | Default | What it controls |
| --- | --- | --- |
| `ZOOM_REMINDER_CRON` | `*/5 * * * *` | How often the class reminder worker runs. |
| `ZOOM_REMINDER_HOURS_BEFORE` | `2` | How many hours ahead the email reminder is sent. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | `*/10 * * * *` | Attendance reconciliation against the Zoom API. |

Reminders and reconciliation need a working Redis (they are BullMQ workers).

## Post-class surveys

If the `mod.surveys` module is active, it can fire NPS surveys automatically at the end of every live class (see `SURVEYS_REMINDER_*` in the [variable reference](variables-de-entorno.md#4-redis-queues-and-workers)).
