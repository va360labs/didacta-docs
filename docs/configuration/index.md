# Configuración

Didacta se configura en **dos capas** complementarias:

## 1. Variables de entorno (instalación)

Definen la infraestructura de la instancia: base de datos, Redis, storage, SMTP global, licencia, telemetría… Se fijan en el `.env` (Docker Compose) o en el entorno del contenedor, y aplican a **toda la instalación**.

→ [Referencia 1 a 1 de todas las variables](environment-variables.md)

Solo 3 son obligatorias (`ADMIN_DATABASE_URL`, `REDIS_URL`, `AUTH_SECRET`); el resto tienen defaults razonables.

## 2. Ajustes por tenant (panel de administración)

Todo lo específico de una organización vive en la base de datos y se gestiona desde **Administración** en la web — nunca en código ni en env vars:

| Ajuste | Dónde |
| --- | --- |
| Idioma de la academia (es, en, pt-BR) | Administración → Configuración → General |
| Marca: logo, colores, textos de acceso | Administración → Branding |
| Página pública de la academia, dominios propios | [Branding](branding.md) |
| Modo sencillo o completo | [Modo sencillo](simple-mode.md) |
| SMTP propio del tenant y «Correo para respuestas» | Administración → Configuración → Notificaciones |
| Plantillas de los correos | Administración → Emails |
| Proveedor y clave de IA (BYOK) | Administración → Proveedores de IA |
| Credenciales Zoom Server-to-Server | Administración, grupo «Integraciones y API» |
| Verificadores de inscripción, bot de Telegram | Administración → Configuración → Registro |
| Registro opt-in con el equipo de Didacta | Administración → Configuración → Registro |

Los secretos por tenant (client secrets OIDC, tokens, claves Stripe, bot de Telegram…) se **cifran at-rest** con la clave maestra de la instancia (`TENANT_SETTINGS_ENC_KEY` o la clave autogenerada en `/app/data/secrets`, dentro del volumen de datos pero fuera del storage que se sirve por HTTP).

!!! warning "Fija la clave de cifrado antes de configurar integraciones"
    Si no defines `TENANT_SETTINGS_ENC_KEY` ni persistes el volumen de datos, en producción la API **no arranca** (y fuera de producción la clave sería efímera y los secretos por tenant se perderían al reiniciar). Haz copia de la clave autogenerada. Detalles en [Variables de entorno → Autenticación y cifrado](environment-variables.md#5-autenticacion-sesiones-y-cifrado).

## Guías por área

- [Base de datos](database.md) — PostgreSQL, pgvector, RLS y migraciones.
- [Almacenamiento](storage.md) — disco local, MinIO o S3.
- [Email](email.md) — SMTP global y por tenant.
- [IA (BYOK)](ai.md) — el AI Gateway multi-proveedor.
- [Aula virtual (Zoom)](zoom.md) — credenciales y webhooks.
- [Branding whitelabel](branding.md) — marca por tenant, página pública y dominios.
- [Idiomas](languages.md) — español, inglés y portugués de Brasil.
- [Modo sencillo](simple-mode.md) — el panel corto para ir de cero a vender.
- [Telemetría](telemetry.md) — qué se envía y cómo desactivarla.
