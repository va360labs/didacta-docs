# Referencia de endpoints

Referencia **endpoint a endpoint** de la API de Didacta, extraída del código real (~100 controllers, ~540 endpoints), organizada en cinco áreas:

<div class="didacta-cards">
  <a href="nucleo/"><h3>🔐 Núcleo y transversales</h3><p>Auth, MFA, API keys, mi cuenta, setup, licencia, branding, storage, auditoría, SSO, SCIM, inscripción externa y webhooks.</p></a>
  <a href="administracion/"><h3>🛠️ Administración</h3><p>Organizaciones, usuarios, invitaciones, módulos, SMTP, identidad corporativa (EE), IA, moderación y Fundae.</p></a>
  <a href="aprendizaje/"><h3>🎓 Aprendizaje</h3><p>Cursos, matrículas y progreso, drip, SCORM, rutas, evaluaciones, certificados, grupos de acceso y Fundae.</p></a>
  <a href="comunidad/"><h3>👥 Comunidad y personas</h3><p>Comunidad, mensajería SSE, gamificación, recursos, encuestas, referidos, theming, inscripción y membresía.</p></a>
  <a href="pagos-directo-ia/"><h3>💳 Pagos, aula virtual e IA</h3><p>Billing, suscripciones, conexiones de pago, Zoom en directo y los tres módulos de IA.</p></a>
</div>

## Cómo leer la referencia

- Todas las rutas cuelgan de **`/api/v1`** salvo las marcadas con ⚠ (`/healthz`, `/metrics`, `/api/license`, `/scim/v2`).
- La columna **Auth** indica el mínimo necesario: `Público` (sin sesión, tenant resuelto por dominio), `Bearer` (cualquier usuario autenticado), `formador+`/`staff`, `admin` (tenant_admin o super_admin), `super_admin`, `ApiKey` + scope, o una firma de webhook.
- Los endpoints Enterprise responden **402** con la capability requerida en el cuerpo; los de un módulo desactivado para el tenant, **403**.
- Los errores de dominio llevan un campo `code` estable (`COURSE_SLUG_EXISTS`, `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING`…) pensado para programar contra él.

!!! tip "El contrato fino, en tu Swagger"
    Esta referencia cubre rutas, auth, propósito, cuerpos principales y errores. Los schemas exactos de request/response de **tu versión desplegada** están siempre en el Swagger de tu instancia: `https://tu-instancia/api/docs` (JSON en `/api/docs.json`).
