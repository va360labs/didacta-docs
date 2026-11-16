# Referencia — Núcleo y transversales

Autenticación, cuenta, setup, licencia pública, branding, sitio público del aula, storage, auditoría, notificaciones, SSO, SCIM, integración externa, agentes y webhooks. Todas las rutas cuelgan de `/api/v1` salvo las marcadas con ⚠.

## Autenticación — `/auth`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/auth/tenant-context` | Público | Resuelve el tenant por el header `Host` y devuelve la marca de las pantallas de acceso (logo, favicon, portada, titulares, color, cifras reales, `membershipPageActive`, `subscriptionGate.state`: `LOCKED` solo si el aula está en pausa) o `{ tenant: null }`. |
| POST | `/auth/signup` | Público | Registra un usuario `alumno` en el tenant (si el registro está habilitado) y devuelve tokens. Acepta `timezone` y `locale` del navegador solo para crear la cuenta. |
| POST | `/auth/signin` | Público | Login email+contraseña → `{ tokens, mfaRequired, user }` (más `mfaRequiredSoon` si la política de MFA está en periodo de gracia). |
| POST | `/auth/refresh` | Público (refresh token en body) | Renueva los tokens (`{ tokens }`) y registra IP/dispositivo en la sesión. |
| POST | `/auth/forgot-password` | Público | Envía email de reset. **Responde 200 siempre** (anti-enumeración). |
| POST | `/auth/reset-password` | Público | Confirma el reset con el token del email + nueva contraseña (12–128). Devuelve `email` y `tenantSlug` para entrar al momento. |
| POST | `/auth/reset-password/status` | Público | Estado de un enlace de acceso **sin consumirlo**: `valid`, `expired`, `used` o `invalid`. |
| POST | `/auth/reset-password/resend` | Público | Con un enlace **caducado** y sin usar, manda otro al correo de esa cuenta (una sola vez por enlace). Responde 200 siempre. |

Errores relevantes: `signin` responde 401 `AUTH_INVALID_CREDENTIALS` («Credenciales inválidas»); si el email existe en varias organizaciones devuelve `AMBIGUOUS_TENANT` con `candidateSlugs` para repetir el login con `tenantSlug`. `signup` responde 403 `AUTH_SIGNUP_DISABLED` si el registro está deshabilitado y 409 `AUTH_EMAIL_TAKEN` si el email ya existe en el tenant. Estas rutas tienen un [límite de intentos propio](../authentication.md#limite-de-intentos-de-acceso) (429 `AUTH_RATE_LIMITED`).

### Acceso de soporte — `/auth/support-access`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/auth/support-access/redeem` | Público (token de un solo uso) | Canjea el token que abrió el plano de control por una sesión de duración limitada, **sin refresh token**. Errores: 400 `SUPPORT_ACCESS_INVALID`; 410 `SUPPORT_ACCESS_EXPIRED` / `_ALREADY_REDEEMED` / `_REVOKED`; 403 `SUPPORT_ACCESS_TENANT_UNAVAILABLE`. |
| GET | `/auth/support-access/current` | Bearer | `{ support: {…} }` si la sesión es de soporte y sigue viva; `{ support: null }` en una sesión normal. Lo consume el aula para su aviso permanente. |

Cómo se abre el acceso, en [Instancia y plano de control](instance-operations.md#acceso-de-soporte).

## MFA — `/auth/mfa`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/auth/mfa/setup` | Bearer | Genera secreto TOTP + QR + códigos de recuperación (aún no activa MFA). |
| POST | `/auth/mfa/enable` | Bearer | Confirma con el primer código de 6 dígitos; audita y reemite tokens con `mfaVerified=true`. |
| POST | `/auth/mfa/verify` | Bearer | Verifica el segundo factor (TOTP o código de recuperación) y eleva la sesión. |

La política de MFA de toda la organización está en [Administración](administration.md#identidad-corporativa-ee) (`/admin/mfa-policy`).

## API keys — `/auth/api-keys`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/auth/api-keys` | admin | Crea una API key: `{ name, scopes[], expiresAt? }`. Los scopes salen de un catálogo cerrado (ver [Autenticación](../authentication.md#scopes)). **El token `lmsk_…` solo se devuelve aquí.** |
| GET | `/auth/api-keys` | admin | Lista las keys propias (sin tokens). Las de todo el aula están en `GET /admin/api-keys`. |
| DELETE | `/auth/api-keys/:id` | admin | Revoca (marca `revokedAt`) → 204. |

Solo `tenant_admin` y `super_admin`: cualquier otro rol recibe 403 `ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`.

## Mi cuenta — `/me`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · PATCH | `/me/profile` | Bearer | Perfil completo · edición (`name`, `bio` ≤280, `jobTitle`, `department`, `location`, `locale` —`es-ES`, `es-AR`, `en-US`, `pt-BR`—, `timezone`, `avatarUrl`, DNI/NIE validado). `''` o `null` borran un campo; omitirlo lo deja igual. |
| GET · POST | `/me/onboarding/status` · `complete` | Bearer | Estado del onboarding (`missing[]`) · marcarlo completado (422 `AUTH_ONBOARDING_MISSING_FIELDS` si faltan nombre o avatar). |
| GET · PUT | `/me/notification-preferences` | Bearer | Matriz categoría × canal (`COMMUNITY\|LEARNING\|ASSESSMENTS\|SYSTEM` × `EMAIL\|IN_APP`). Sin fila, el valor por defecto es activado. |
| POST | `/me/security/password` | Bearer | Cambia la contraseña verificando la actual; **cierra todas las sesiones**. |
| GET | `/me/security/sessions` | Bearer | Hasta 20 sesiones activas (fechas, IP, user agent). |
| DELETE | `/me/security/sessions/:id` | Bearer | Cierra una sesión concreta (efecto inmediato). |
| POST | `/me/security/sessions/revoke-others` | Bearer | Cierra todas las sesiones. |
| GET | `/me/modules` | Bearer | Módulos activos del tenant + capabilities Enterprise (alimenta el menú); a los administradores les trae además el estado de la puerta de suscripción. |
| GET | `/me/purchases` | Bearer | Mis compras hechas en la tienda externa del centro (las que esa tienda dejó con `POST /integrations/orders`): importe y, si existe, número de factura y enlace al PDF que sirve quien la emitió. Vacío si no vende una tienda externa. Didacta no genera facturas aquí. |

## Perfiles públicos — `/users`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/users/public?ids=a,b,c` | Bearer | Batch de `{ id, name, avatarUrl }` (máx. 100) para avatares del feed. |
| GET | `/users/:id/public-profile` | Bearer | Perfil público (nombre, avatar, cargo, departamento, ubicación, bio). Nunca email, DNI ni roles. |

## Notificaciones — `/me/notifications`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/me/notifications` | Bearer | Mis notificaciones in-app (máx. 100, recientes primero). |
| POST | `/me/notifications/:id/read` · `read-all` | Bearer | Marca como leída / todas (idempotente). |
| POST | `/me/notifications/stream-ticket` | Bearer | Emite un ticket JWT de 60 s para el stream. |
| GET | `/me/notifications/stream?ticket=` | Ticket SSE | **SSE** — stream en tiempo real (`notification` / `ping`). |

## Setup — `/setup`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/setup/status` | Público | ¿La instancia ya tiene organización? |
| GET | `/setup/available-modules` | Público | Módulos para el asistente (`isCore`, `enabledByDefault`). |
| POST | `/setup/init` | Público | Bootstrap del primer arranque: organización (`name`, `slug?`, `primaryHostname?`, `locale?`), primer `super_admin` (`name`, `email`, `password`) y `enabledModules?`. Devuelve tokens. 409 `ALREADY_INITIALIZED` si ya hay tenants. |

## Plataforma

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | ⚠ `/api/license` | Público | Estado público de la licencia (status, capabilities, avisos). Exento de rate limit y de la puerta de suscripción. La gestión de la licencia (`/admin/license`) está en [Instancia y plano de control](instance-operations.md#licencia-adminlicense-super_admin). |
| GET | `/branding/options` | Público | Branding del tenant para la UI (`logoUrl`, `primaryColor`, `poweredByDidacta`). |
| GET · POST | `/branding/white-label/preview` · `configure` | super_admin <span class="didacta-chip didacta-chip--ee">EE</span> `feat:white_label` | Estado y configuración white-label. **402** sin licencia, 403 si no eres super_admin. |
| GET | `/system/version-check` | Público | Proxy a los tags de Docker Hub para el banner de «versión nueva» (cache 15 min). |
| GET | ⚠ `/healthz` · `/livez` | Público | Liveness (versión, uptime). |
| GET | ⚠ `/readyz` | Público | Readiness: comprueba BD, Redis y storage; **503** si algo está degradado. |
| GET | ⚠ `/metrics` | `Bearer <METRICS_TOKEN>` | Métricas Prometheus/OpenMetrics. Con `METRICS_TOKEN` definido exige ese token; **en producción sin `METRICS_TOKEN` responde 401**: defínelo en el entorno de la API. |

## Sitio público del aula (sin sesión)

Endpoints que usa la web para pintar lo que ve un visitante. El aula sale del dominio de la petición (`X-Forwarded-Host`, o `Host`); un dominio que no es de ningún aula activo responde 404.

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/tenancy/resolve` | Público | `{ known, slug, host, surface, suspended }`: a qué aula corresponde el dominio y si sirve la app (`APP`) o el sitio público (`SITE`). Un dominio de aula suspendida da `known: false` y `suspended: true`. |
| GET | `/tenancy/locale` | Público | `{ locale: "es" \| "en" \| "pt" \| null }`: idioma del aula del dominio. |
| GET | `/public/academy-home` | Público | Portada pública: marca, nº de miembros, quién la lleva, próximo directo y opciones de acceso (cursos a la venta, membresía, acceso gratuito). `{ enabled: false }` si el aula la tiene apagada. |
| GET | `/public/seo` | Público | Indexación: si se indexa, dominio canónico, idioma, datos públicos y fichas de venta con su última modificación. `{ indexable: false }` con la página pública apagada. |
| GET | `/public/site-context` | Público | Tenant, módulos activos y origen canónico del sitio público. 404 si el dominio no sirve sitio público. |
| GET | `/public/checkout-seller` | Público | Quién vende en el aula: nombre, quien la lleva y el correo de contacto configurado. |
| GET | `/purchase-consent/withdrawal-waiver` | Público | Texto y versión de la casilla «quiero empezar ya y pierdo el desistimiento» para esta aula, en el idioma del comprador. |

## Storage — `/storage`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/storage/upload` | alumno, formador o admin | Sube imagen o documento en base64 (máx. 10 MiB, MIME de lista cerrada); optimiza imágenes a WebP. |
| POST | `/storage/optimize` | formador+ | Reoptimiza una imagen ya subida y devuelve la nueva URL. |
| GET | `/storage/file/*` | Público | Sirve un fichero del storage local por su key (con `CSP: sandbox` y `nosniff`). Con S3 se usan URLs prefirmadas. |

## Ajustes de tenant — `/tenant-settings` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/tenant-settings[/:scope[/:key]]` | Lista/lee settings; los secretos devuelven los campos no sensibles y redactan credenciales. |
| PUT | `/tenant-settings/:scope/:key` | Crea/actualiza (`{ value, isSecret }`); cifrado at-rest si es secreto. |
| DELETE | `/tenant-settings/:scope/:key` | Elimina el setting. |
| POST | `/tenant-settings/notifications/smtp/test` | Email de prueba con la config SMTP del tenant. |

Los ajustes con pantalla propia (SMTP, Stripe, academia) tienen endpoints dedicados en [Administración](administration.md).

## Auditoría — `/audit` (admin o auditor)

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/audit/entries` | admin/auditor | Log de auditoría con filtros (`actorId`, `action`, `resourceType`, fechas). En Community el rango se trunca a 90 días; con `feat:audit.long_retention` es ilimitado. |
| GET | `/audit/verify` | admin/auditor | Verifica la integridad de la cadena hash del log. |
| GET | `/audit/retention-info` | admin/auditor | Política activa: `{ plan, maxDays, capability }`. |
| GET | `/audit/entries.zip?from=&to=` | admin <span class="didacta-chip didacta-chip--ee">EE</span> `feat:reports.advanced_signed` | Export ZIP **firmado** (manifest + NDJSON + firma), verificable offline. **402** sin capability. |

## SSO (flujos públicos)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/auth/oidc/:tenantSlug/status` · `start` · `/auth/oidc/callback` | OIDC: ¿habilitado? · redirección al IdP (state+nonce+PKCE) · callback que emite la sesión y redirige al frontend. |
| GET/POST | `/auth/saml/:tenantSlug/status` · `login` · `acs` · `metadata` | SAML 2.0: estado · AuthnRequest · Assertion Consumer Service (form-urlencoded) · metadata XML del SP. |
| GET | `/modules/wp-sso/:tenantSlug/status` · `callback?token=` | WP-SSO: config pública · intercambio del token HMAC de WordPress por sesión Didacta (302). |

Sin configuración habilitada, los flujos responden 404. Los errores siempre redirigen a `/auth/error?reason=…` con códigos legibles (`state_expired`, `email_not_allowed`, `user_not_provisioned`…).

## SCIM 2.0 — ⚠ `/scim/v2` (Bearer SCIM propio)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/scim/v2/ServiceProviderConfig` · `ResourceTypes` · `Schemas` | Discovery (no gateado). |
| GET · POST | `/scim/v2/Users` | Lista con paginación SCIM y filtro `userName eq` · crea usuario (201; 409 `uniqueness` si el `userName` ya existe). |
| GET · PATCH · DELETE | `/scim/v2/Users/:id` | Lee · aplica `PatchOp` (active, name, locale…) · soft-delete (204). |
| ANY | `/scim/v2/Groups[/*]` | **501** con cuerpo de error SCIM: Groups no está implementado. |

Los cinco CRUD de usuarios requieren la capability `feat:scim` (**402** sin licencia). El token se emite en `/admin/scim/token`. Guía completa —mapeo de atributos, formato de error y lo que **no** está soportado— en [Aprovisionamiento SCIM](../../enterprise/scim.md).

## Inscripción externa — `/inscribe` (API keys)

| Método | Ruta | Scope | Qué hace |
|---|---|---|---|
| POST | `/inscribe` | `enrollments:write` | Crea-o-reusa usuario por email (contraseña temporal por correo) y lo matricula en `courseIds`/`accessGroupIds` (al menos uno). Acepta `name`, `locale` y `externalRef`. Idempotente. |
| POST | `/inscribe/revoke` | `enrollments:write` | Baja de matrículas de origen API (reembolso). Email inexistente → `userFound: false`, no 404. No revoca accesos por grupo, suscripción o compra directa. |
| GET | `/inscribe/courses` | `courses:read` | Catálogo con `status`, para mapear producto → curso. |
| GET | `/inscribe/access-groups` | `courses:read` | Grupos de acceso con `kind` y `courseCount`. |

Auth: `Authorization: ApiKey lmsk_…`. Un JWT de usuario no pasa estos endpoints: exigen scopes de API key.

## Lectura de fichas y compras para sitios externos — `/integrations` (API keys)

La otra mitad de `/inscribe`: aquella escribe matrículas, esta lee fichas y guarda el historial de compra. Pensada para que un sitio de fuera pinte la ficha de un curso con datos de Didacta, sepa si quien la mira ya es alumno y deje sus pedidos en su perfil.

| Método | Ruta | Scope | Qué hace |
|---|---|---|---|
| GET | `/integrations/courses` | `courses:read` | Cursos filtrables por `slug`, por `status` o por `externalId` + `externalSource` (mapeo desde el LMS de origen). |
| GET | `/integrations/courses/:courseId` | `courses:read` | Ficha completa: metadatos, formador, totales, **temario módulo a módulo** y oferta de compra. **Nunca** el `content` de una lección. |
| GET | `/integrations/learners/state` | `enrollments:read` | Estado de un alumno por `email` (y `courseId?`): matrículas, progreso, `nextLesson` con URL directa y `membership` viva (o `null`). `known:false` si el email no es de nadie. |
| POST | `/integrations/orders` | `orders:write` | Guarda en el perfil del alumno una compra hecha en una tienda externa. **Idempotente por `(source, reference)`**; `invoice`, `orderUrl` y `refundedAt` solo se escriben si vienen (omitirlos no borra; `invoice: null` retira la factura). Didacta no emite ni numera facturas. |
| GET | `/integrations/learners/orders` | `orders:read` | Historial de compra de un alumno por `email`. `known: false` no implica lista vacía. |

`enrollments:read` va separado de `enrollments:write` a propósito: permite preguntar por **cualquier** email de la organización.

Ojo con `hasAccess:false` + `status:"PAUSED"`: es un alumno **suspendido** (típicamente por impago) con su progreso intacto, no alguien que nunca compró.

Guía completa con ejemplos: [Vender e integrar desde fuera](../external-integrations.md).

## Agentes de IA — `/agent` y `/mcp`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/agent/keys` | admin | Crea la clave de un agente de IA. El servidor fija los scopes (`agent:courses:read`, `agent:courses:write`, `agent:live:write`, `agent:community:post`); el cliente no los elige. **El token solo se devuelve aquí.** |
| POST | `/mcp` | Clave de agente (`Bearer lmsk_…` o `ApiKey lmsk_…`) | Servidor MCP del aula (Streamable HTTP, JSON-RPC 2.0). `GET` y `DELETE` sobre `/mcp` responden 405 (sin stream ni sesiones). |

Herramientas, límites por clave y lo que un agente **no** puede hacer: [MCP](../mcp.md).

## Webhooks salientes — `/webhooks` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/webhooks/info` | Tier activo, límites efectivos y catálogo de eventos suscribibles. |
| GET | `/webhooks/endpoints[/:id]` | Lista / detalle (secret enmascarado). |
| POST | `/webhooks/endpoints` | Crea: `{ url (https), eventTypes[] («*» = todos), secret?, active? }`. Secret en claro **una sola vez**. |
| PUT | `/webhooks/endpoints/:id` | Actualiza; enviar `secret` lo rota (one-shot). |
| DELETE | `/webhooks/endpoints/:id` | Elimina (204, idempotente). |

Errores: 409 URL duplicada · 422 `webhook_limit_exceeded` al superar el límite del plan (Community: 1 endpoint / 3 tipos de evento; Enterprise: 20 / ilimitados). La cola de reintentos fallidos (dead-letter, EE) está en [Administración](administration.md#identidad-corporativa-ee). El formato de la entrega, el catálogo de eventos y la firma, en [Webhooks](../webhooks.md).
