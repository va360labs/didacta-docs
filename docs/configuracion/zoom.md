# Aula virtual (Zoom)

Las clases en directo de Didacta usan **Zoom** (API Server-to-Server + SDK Web) a través del módulo `mod.zoom-live`: sesiones, inscripción con enlace compartible, acceso al `joinUrl` y a la grabación controlado por matrícula, y asistencia reconciliada contra la API de Zoom.

## Credenciales: por tenant, no por entorno

Las credenciales **Server-to-Server OAuth** de Zoom (Account ID, Client ID, Client Secret) **no son variables de entorno**: se configuran por organización en el panel de administración y se guardan **cifradas at-rest** (de ahí la importancia de [`TENANT_SETTINGS_ENC_KEY`](variables-de-entorno.md#5-autenticacion-sesiones-y-cifrado)).

Necesitas una app *Server-to-Server OAuth* creada en el [Zoom App Marketplace](https://marketplace.zoom.us/) de tu cuenta, con permisos de reuniones y grabaciones.

## Webhook de Zoom

Para que Didacta reciba eventos (inicio/fin de reunión, grabaciones disponibles, asistencia):

1. En tu app de Zoom, configura el endpoint de webhook apuntando a tu instancia:
   ```
   https://tu-dominio/api/v1/webhooks/zoom
   ```
2. Copia el **Secret Token** de Zoom en la variable:
   ```bash
   ZOOM_WEBHOOK_SECRET=...
   ```

Didacta verifica la firma HMAC-SHA256 de cada evento contra el cuerpo crudo de la petición; sin el secret, los eventos se rechazan.

## Trabajos programados

| Variable | Default | Qué controla |
| --- | --- | --- |
| `ZOOM_REMINDER_CRON` | `*/5 * * * *` | Frecuencia del worker de recordatorios de clase. |
| `ZOOM_REMINDER_HOURS_BEFORE` | `2` | Horas de antelación del recordatorio por email. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | `*/10 * * * *` | Reconciliación de asistencia contra la API de Zoom. |

Los recordatorios y la reconciliación requieren Redis operativo (son workers BullMQ).

## Encuestas post-clase

Si el módulo `mod.surveys` está activo, puede disparar encuestas NPS automáticamente al terminar cada clase en directo (ver `SURVEYS_REMINDER_*` en la [referencia de variables](variables-de-entorno.md#4-redis-colas-y-workers)).
