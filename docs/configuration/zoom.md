# Aula virtual (Zoom)

Las clases en directo de Didacta usan **Zoom** (API Server-to-Server + SDK Web) a través del módulo `mod.zoom-live`: sesiones, inscripción con enlace compartible, acceso al `joinUrl` y a la grabación controlado por matrícula, y asistencia reconciliada contra la API de Zoom.

## Credenciales: por tenant, no por entorno

Las credenciales **Server-to-Server OAuth** de Zoom (Account ID, Client ID, Client Secret) **no son variables de entorno**: se configuran por organización en el panel de administración y se guardan **cifradas at-rest** (de ahí la importancia de [`TENANT_SETTINGS_ENC_KEY`](environment-variables.md#5-autenticacion-sesiones-y-cifrado)).

Necesitas una app *Server-to-Server OAuth* creada en el [Zoom App Marketplace](https://marketplace.zoom.us/) de tu cuenta, con permisos de reuniones y grabaciones.

## Webhook de Zoom

Para que Didacta reciba eventos (inicio/fin de reunión, grabaciones disponibles, asistencia):

1. En tu app de Zoom, configura el endpoint de webhook apuntando al **dominio de tu academia** (la URL exacta aparece, con botón «Copiar», en el bloque «Webhook de Zoom» de la pestaña **Aula virtual**):
   ```
   https://tu-dominio/api/v1/webhooks/zoom
   ```
2. Pega el **Secret Token** de Zoom en el campo **Webhook Secret Token** de **Administración → Configuración → Aula virtual**, junto a tus credenciales (**no** es una variable de entorno: es **por academia**, cifrado) y pulsa **«Comprobar webhook»**.

Didacta verifica la firma HMAC-SHA256 de cada evento contra el cuerpo crudo de la petición con el secreto **de la academia a la que pertenece el dominio** de la URL; sin él, los eventos se rechazan, y un evento nunca se procesa para un aula distinta de la que firmó. Una misma app de Zoom repartida entre varias academias del mismo dueño funciona si todas guardan el mismo secreto. Los avisos recibidos se ven en `/admin/webhooks?tab=zoom`. Detalle en [mod.zoom-live](../modules/catalog/zoom-live.md).

## Trabajos programados

| Variable | Default | Qué controla |
| --- | --- | --- |
| `ZOOM_REMINDER_CRON` | `*/5 * * * *` | Frecuencia del worker de recordatorios de clase. |
| `ZOOM_REMINDER_HOURS_BEFORE` | `2` | Horas de antelación del recordatorio por email. |
| `ZOOM_ATTENDANCE_SYNC_CRON` | `*/10 * * * *` | Reconciliación de asistencia contra la API de Zoom. |

Los recordatorios y la reconciliación requieren Redis operativo (son workers BullMQ).

## Encuestas post-clase

Si el módulo `mod.surveys` está activo, puede disparar encuestas NPS automáticamente al terminar cada clase en directo (ver `SURVEYS_REMINDER_*` en la [referencia de variables](environment-variables.md#4-redis-colas-y-workers)).
