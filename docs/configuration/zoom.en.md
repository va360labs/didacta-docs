# Virtual classroom (Zoom)

Didacta's live classes use **Zoom** (Server-to-Server API + Web SDK) through the `mod.zoom-live` module: sessions, registration with a shareable link, access to the `joinUrl` and the recording gated by enrollment, and attendance reconciled against the Zoom API.

## Credentials: per tenant, not per environment

Zoom's **Server-to-Server OAuth** credentials (Account ID, Client ID, Client Secret) are **not environment variables**: they are configured per organization in the admin panel and stored **encrypted at rest** (hence the importance of [`TENANT_SETTINGS_ENC_KEY`](environment-variables.md#5-authentication-sessions-and-encryption)).

You need a *Server-to-Server OAuth* app created in your account's [Zoom App Marketplace](https://marketplace.zoom.us/), with meeting and recording scopes.

## Zoom webhook

So that Didacta receives events (meeting start/end, recordings available, attendance):

1. In your Zoom app, point the webhook endpoint at **your academy's domain** (the exact URL appears, with a "Copy" button, in the "Zoom webhook" block of the **Virtual classroom** tab):
   ```
   https://your-domain/api/v1/webhooks/zoom
   ```
2. Paste Zoom's **Secret Token** into the **Webhook Secret Token** field under **Administration → Settings → Virtual classroom**, next to your credentials (it is **not** an environment variable: it is **per academy**, encrypted) and click **"Comprobar webhook"** (Check webhook).

Didacta verifies the HMAC-SHA256 signature of every event against the raw request body, using the secret **of the academy the URL's domain belongs to**; without it, events are rejected, and an event is never processed for an academy other than the one that signed it. One Zoom app shared by several academies of the same owner works if all of them store the same secret. Received notices are listed at `/admin/webhooks?tab=zoom`. Details in [mod.zoom-live](../modules/catalog/zoom-live.md).

## Scheduled jobs

| Variable | Default | What it controls |
| --- | --- | --- |
| `ZOOM_REMINDER_CRON` | `*/5 * * * *` | How often the class reminder worker runs. |
| `ZOOM_REMINDER_HOURS_BEFORE` | `2` | How many hours ahead the email reminder is sent. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | `*/10 * * * *` | Attendance reconciliation against the Zoom API. |

Reminders and reconciliation need a working Redis (they are BullMQ workers).

## Post-class surveys

If the `mod.surveys` module is active, it can fire NPS surveys automatically at the end of every live class (see `SURVEYS_REMINDER_*` in the [variable reference](environment-variables.md#4-redis-queues-and-workers)).
