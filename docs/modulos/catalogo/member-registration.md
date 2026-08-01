# mod.member-registration — Inscripción de miembros

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Alta de miembros con **validación manual**: solicitud → evidencia → decisión humana. El wizard público (`/inscripcion-miembros`) monta sus pasos según la política del tenant:

- Verificación por **Telegram** (Login Widget + comprobación de pertenencia al grupo).
- **OTP por email** (código de 6 dígitos).
- Ambas, o **registro libre** (el único gate es la aprobación manual).

El registro crea el usuario en `PENDING`, lanza en segundo plano el **lookup de suscripciones y compras** del email en las cuentas de pago conectadas, y avisa al **aprobador** por email con dos enlaces firmados de un solo uso (aprobar / rechazar) más el estado de impago si consta.

## Cómo funciona

- Los pasos se encadenan con **tickets firmados HMAC** (sin estado en base de datos): el ticket de Telegram autoriza el OTP y el `verificationToken` del OTP autoriza el registro.
- Aprobar activa al usuario, le asigna el **grupo de acceso por defecto** y envía la bienvenida; rechazar lo desactiva y avisa.
- Si el tenant exige un verificador no operativo (p. ej. Telegram sin bot configurado), la inscripción responde **503 fail-closed** — nunca deja pasar sin verificar.
- Un worker diario de **purga GDPR** anonimiza los datos de Telegram de las solicitudes nunca decididas al vencer la retención (90 días por defecto).
- El panel admin permite además el **alta manual** sin OTP, re-lanzar el lookup de pagos, decidir sin el email, enviar recordatorios de renovación y gestionar **flags de impago** (con import CSV).

## Dependencias

Duras: `mod.payment-connections` (lookup y enlaces de renovación) y `mod.access-groups` (grupo por defecto al aprobar).

## Modelo de datos

`mod_member_registration_profile` (datos del flujo por usuario) · `mod_member_registration_decision_token` (tokens de un solo uso, hash SHA-256, TTL 7 días) · `mod_member_registration_payment_flag` (impagos de referencia) · `mod_member_registration_subscription_lookup` (resultado del lookup).

## API

Prefijo `/modules/member-registration` (bloques público, admin y payment-flags). Detalle en [Referencia → Comunidad y personas](../../api/referencia/comunidad.md#inscripcion-de-miembros-modulesmember-registration).

## Eventos

**Emite**: `member_registration.request.created/approved/rejected`. No consume.

## Configuración

- **Por tenant** (Administración → Configuración → Registro): bot de Telegram (token cifrado at-rest), verificadores exigidos y email del aprobador. Registra 4 plantillas de email editables en Administración → Emails.
- **Fallback global legacy** para mono-tenant: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_GROUP_ID`, `TELEGRAM_BOT_USERNAME`, `MEMBER_APPROVAL_EMAIL`.
- Workers: `MEMBER_PURGE_CRON` (03:00 UTC), `MEMBER_RETENTION_DAYS` (90).
