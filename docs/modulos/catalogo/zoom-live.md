# mod.zoom-live — Aula virtual (Zoom)

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **live** (desactivable)

## Qué hace

Clases en directo con la API **Server-to-Server OAuth** de Zoom. El formador programa sesiones (opcionalmente vinculadas a un curso o a una lección concreta, lo que habilita el botón «Unirse» en la lección); los alumnos se inscriben desde un enlace compartible, y el `joinUrl` y la **grabación** quedan gateados server-side a inscritos o staff. Incluye:

- **Añadir al calendario** al inscribirse: Google, Outlook.com, Microsoft 365 y `.ics` (los enlaces nunca contienen el `joinUrl`).
- **Recordatorio** por email + in-app 2 horas antes (uno solo por clase, con claim atómico anti-duplicados).
- **Asistencia real**: proxy de entrada (clic en «Unirme») reconciliado después contra la Report API de Zoom, con override manual del formador.
- La grabación se persiste al llegar el webhook `recording.completed`.

## Cómo funciona

- Sin credenciales configuradas, el módulo usa un **stub determinístico** — se puede probar todo el flujo sin cuenta de Zoom.
- Los webhooks se procesan con **log idempotente** por `event_id` (Zoom reintenta hasta 3 veces) y firma HMAC verificada; incluye el handshake de validación de endpoint de Zoom.
- Los participantes de Zoom que no casan con ningún usuario (invitados, otro email) se conservan con su identidad de Zoom para conciliación manual.
- El error típico del sync de asistencia es que falte el scope `report:read:admin` en la app de Zoom; queda registrado en la sesión para que el formador sepa por qué no hay datos.
- Al terminar la clase, `mod.surveys` puede lanzar automáticamente la encuesta post-clase.

## Dependencias

Opcional: `mod.courses` (vincular sesión ↔ curso/lección).

## Modelo de datos

`mod_zoom_session` (clase: horario, host, URLs, grabación, marcas de sync) · `_session_registration` (inscripción, base del gating) · `_session_attendance` (clic de entrada + datos de Zoom + override) · `_webhook_event` (log idempotente).

## API

Prefijo `/modules/zoom-live` (+ calendario público + webhook en `/webhooks/zoom`). Detalle en [Referencia → Pagos, aula e IA](../../api/referencia/pagos-directo-ia.md#aula-virtual-moduleszoom-live).

## Eventos

**Emite**: `zoom.session.created/updated/cancelled/started/ended`, `zoom.session.recording_ready`, `zoom.session.registration.created/cancelled`, `zoom.session.attendance_synced`. No consume.

## Configuración

- **Por tenant** (panel): credenciales S2S de Zoom (`accountId`, `clientId`, `clientSecret`), cifradas at-rest.
- **Entorno**: `ZOOM_WEBHOOK_SECRET`, `ZOOM_REMINDER_CRON`, `ZOOM_REMINDER_HOURS_BEFORE`, `ZOOM_ATTENDANCE_SYNC_CRON`. Ver [Configurar Zoom](../../configuracion/zoom.md).
