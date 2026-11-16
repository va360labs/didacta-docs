# Referencia de endpoints

Referencia **endpoint a endpoint** de la API de Didacta, extraída del código real de la versión 0.1.0-beta.24 (unos 130 controladores y más de 700 endpoints), organizada en seis áreas:

<div class="didacta-cards">
  <a href="core/"><h3>🔐 Núcleo y transversales</h3><p>Auth, MFA, API keys, mi cuenta, setup, licencia pública, sitio público del aula, storage, auditoría, SSO, SCIM, inscripción externa, agentes MCP y webhooks.</p></a>
  <a href="administration/"><h3>🛠️ Administración</h3><p>Usuarios, invitaciones, academia, modo sencillo, módulos, ventas, correos de venta, SMTP, Stripe, identidad corporativa (EE), IA, moderación y Fundae.</p></a>
  <a href="instance-operations/"><h3>🧭 Instancia y plano de control</h3><p>Organizaciones, credencial de provisioning, accesos de soporte, puerta de suscripción, licencia, ajustes de instancia y rutas de Cloud.</p></a>
  <a href="learning/"><h3>🎓 Aprendizaje</h3><p>Cursos, matrículas y progreso, drip, SCORM, rutas, evaluaciones, certificados, grupos de acceso, vídeo y Fundae.</p></a>
  <a href="community/"><h3>👥 Comunidad y personas</h3><p>Comunidad, mensajería SSE, gamificación y retención, recursos, encuestas, referidos, theming, inscripción, membresía y ayuda y sugerencias.</p></a>
  <a href="payments-live-ai/"><h3>💳 Pagos, aula e IA</h3><p>Billing, suscripciones, conexiones de pago, cobros de plataforma (Cloud), Zoom en directo y los tres módulos de IA.</p></a>
</div>

## Cómo leer la referencia

- Todas las rutas cuelgan de **`/api/v1`** salvo las marcadas con ⚠ (`/healthz`, `/metrics`, `/api/license`, `/scim/v2`).
- La columna **Auth** indica el mínimo necesario: `Público` (sin sesión, tenant resuelto por dominio), `Bearer` (cualquier usuario autenticado), `formador+`/`staff`, `admin` (tenant_admin o super_admin), `super_admin`, `super_admin o provisioning` (también acepta la [credencial de provisioning](../authentication.md#3-credencial-de-provisioning)), `ApiKey` + scope, clave de agente, o una firma de webhook.
- Marcas de edición: <span class="didacta-chip didacta-chip--ee">EE</span> = exige una capability Enterprise (sin licencia, **402** con la capability en el cuerpo). **Cloud** = solo existe cuando el operador enciende la función (modo plataforma de cobros, subida de vídeo); sin ella la ruta responde 404.
- Los endpoints de un módulo desactivado para el tenant responden **403** `TENANT_MODULES_MODULE_NOT_ACTIVE`; con el aula en pausa (puerta de suscripción), **402** `SUBSCRIPTION_LOCKED`. Más en [Convenciones](../conventions.md).
- Los errores de dominio llevan un campo `code` estable (`COURSE_SLUG_EXISTS`, `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING`…) pensado para programar contra él.

!!! tip "El contrato fino, en tu Swagger"
    Esta referencia cubre rutas, auth, propósito, cuerpos principales y errores. Los schemas exactos de request/response de **tu versión desplegada** están siempre en el Swagger de tu instancia: `https://tu-instancia/api/docs` (JSON en `/api/docs.json`).

!!! note "Qué no cubre"
    Los módulos de terceros instalados desde el marketplace publican sus propias rutas bajo `/modules/<slug>/…` y se documentan en su paquete. Las tareas de mantenimiento de la instalación (rotar la clave de cifrado, reparar accesos tras reembolsos) son comandos de línea de órdenes, no endpoints.
