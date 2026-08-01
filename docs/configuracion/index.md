# Configuración

Didacta se configura en **dos capas** complementarias:

## 1. Variables de entorno (instalación)

Definen la infraestructura de la instancia: base de datos, Redis, storage, SMTP global, licencia, telemetría… Se fijan en el `.env` (Docker Compose) o en el entorno del contenedor, y aplican a **toda la instalación**.

→ [Referencia 1 a 1 de todas las variables](variables-de-entorno.md)

Solo 3 son obligatorias (`DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`); el resto tienen defaults razonables.

## 2. Ajustes por tenant (panel de administración)

Todo lo específico de una organización vive en la base de datos y se gestiona desde **Administración** en la web — nunca en código ni en env vars:

| Ajuste | Dónde |
| --- | --- |
| Marca: logo, colores, textos de acceso | Administración → Marca |
| SMTP propio del tenant | Administración → SMTP |
| Proveedor y clave de IA (BYOK) | Administración → IA |
| Credenciales Zoom Server-to-Server | Administración → Integraciones |
| Verificadores de inscripción, bot de Telegram | Administración → Configuración → Registro |
| Registro opt-in con el equipo de Didacta | Administración → Registro |

Los secretos por tenant (client secrets OIDC, tokens, claves Stripe, bot de Telegram…) se **cifran at-rest** con la clave maestra de la instancia (`TENANT_SETTINGS_ENC_KEY` o la clave autogenerada en el volumen de datos).

!!! warning "Fija la clave de cifrado antes de configurar integraciones"
    Si no defines `TENANT_SETTINGS_ENC_KEY` ni persistes el volumen de datos, la clave de cifrado será efímera y los secretos por tenant se perderán al reiniciar. Detalles en [Variables de entorno → Autenticación y cifrado](variables-de-entorno.md#5-autenticacion-sesiones-y-cifrado).

## Guías por área

- [Base de datos](base-de-datos.md) — PostgreSQL, pgvector, RLS y migraciones.
- [Almacenamiento](almacenamiento.md) — disco local, MinIO o S3.
- [Email](email.md) — SMTP global y por tenant.
- [IA (BYOK)](ia.md) — el AI Gateway multi-proveedor.
- [Aula virtual (Zoom)](zoom.md) — credenciales y webhooks.
- [Branding whitelabel](branding.md) — marca por tenant y dominios.
- [Telemetría](telemetria.md) — qué se envía y cómo desactivarla.
