# Referencia — Núcleo y transversales

Autenticación, cuenta, setup, licencia, branding, storage, auditoría, notificaciones, SSO, SCIM, inscripción externa y webhooks. Todas las rutas cuelgan de `/api/v1` salvo las marcadas con ⚠.

## Autenticación — `/auth`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/auth/tenant-context` | Público | Resuelve el tenant por el header `Host` y devuelve la marca del panel de acceso (logo, titulares, stats) o `{ tenant: null }`. |
| POST | `/auth/signup` | Público | Registra un usuario en el tenant (si el registro está habilitado) y devuelve tokens. |
| POST | `/auth/signin` | Público | Login email+contraseña → tokens + `mfaRequired` + datos del usuario. |
| POST | `/auth/refresh` | Público (refresh token en body) | Renueva el access token y registra IP/dispositivo en la sesión. |
| POST | `/auth/forgot-password` | Público | Envía email de reset. **Responde 200 siempre** (anti-enumeración). |
| POST | `/auth/reset-password` | Público | Confirma el reset con el token del email + nueva contraseña (12–128). |

Errores relevantes: `signin` responde 401 genérico («Credenciales inválidas»); si el email existe en varias organizaciones devuelve `AMBIGUOUS_TENANT` con los slugs candidatos para repetir el login con `tenantSlug`. `signup` responde 403 si el registro está deshabilitado y 409 si el email ya existe en el tenant.

## MFA — `/auth/mfa`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/auth/mfa/setup` | Bearer | Genera secreto TOTP + QR + códigos de recuperación (aún no activa MFA). |
| POST | `/auth/mfa/enable` | Bearer | Confirma con el primer código de 6 dígitos; audita y reemite tokens con `mfaVerified=true`. |
| POST | `/auth/mfa/verify` | Bearer | Verifica el segundo factor (TOTP o código de recuperación) y eleva la sesión. |

## API keys — `/auth/api-keys`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/auth/api-keys` | Bearer | Crea una API key: `{ name, scopes[], expiresAt? }`. **El token `lmsk_…` solo se devuelve aquí.** |
| GET | `/auth/api-keys` | Bearer | Lista las keys del usuario (sin tokens). |
| DELETE | `/auth/api-keys/:id` | Bearer | Revoca (marca `revokedAt`) → 204. |

## Mi cuenta — `/me`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET · PATCH | `/me/profile` | Bearer | Perfil completo · edición (nombre, bio, cargo, locale, timezone, avatar, DNI/NIE validado). |
| GET · POST | `/me/onboarding/status` · `complete` | Bearer | Estado del onboarding (`missing[]`) · marcarlo completado (422 si faltan campos). |
| GET · PUT | `/me/notification-preferences` | Bearer | Matriz categoría × canal (`COMMUNITY\|LEARNING\|ASSESSMENTS\|SYSTEM` × `EMAIL\|IN_APP`). |
| POST | `/me/security/password` | Bearer | Cambia la contraseña verificando la actual; **cierra todas las sesiones**. |
| GET | `/me/security/sessions` | Bearer | Hasta 20 sesiones activas (fechas, IP, user agent). |
| DELETE | `/me/security/sessions/:id` | Bearer | Cierra una sesión concreta (efecto inmediato). |
| POST | `/me/security/sessions/revoke-others` | Bearer | Cierra todas las sesiones. |
| GET | `/me/modules` | Bearer | Módulos activos del tenant + capabilities Enterprise (alimenta el menú). |

## Perfiles públicos — `/users`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/users/public?ids=a,b,c` | Bearer | Batch de `{ id, name, avatarUrl }` (máx. 100) para avatares del feed. |
| GET | `/users/:id/public-profile` | Bearer | Perfil público (nombre, avatar, cargo, bio). Nunca email, DNI ni roles. |

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
| POST | `/setup/init` | Público | Bootstrap del primer arranque. 409 `ALREADY_INITIALIZED` si ya hay tenants. |

## Plataforma

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | ⚠ `/api/license` | Público | Estado público de la licencia (status, capabilities, avisos). Exento de rate limit. |
| GET | `/branding/options` | Público | Branding del tenant para la UI (`logoUrl`, `primaryColor`, `poweredByDidacta`). |
| GET · POST | `/branding/white-label/preview` · `configure` | Capability `feat:white_label` | Estado y configuración white-label. **402** sin licencia. |
| GET | `/system/version-check` | Público | Proxy a los tags de Docker Hub para el banner de «versión nueva» (cache 15 min). |
| GET | ⚠ `/healthz` · `/livez` | Público | Liveness (versión, uptime). |
| GET | ⚠ `/readyz` | Público | Readiness: comprueba BD, Redis y storage; **503** si algo está degradado. |
| GET | ⚠ `/metrics` | `Bearer <METRICS_TOKEN>` si está definido | Métricas Prometheus/OpenMetrics. |

## Storage — `/storage`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| POST | `/storage/upload` | Bearer (cualquier rol) | Sube imagen o documento en base64 (máx. 10 MiB, MIME de lista cerrada); optimiza imágenes a WebP. |
| POST | `/storage/optimize` | Bearer (formador+) | Reoptimiza una imagen ya subida y devuelve la nueva URL. |
| GET | `/storage/file/*` | Público | Sirve un fichero del storage local por su key (con `CSP: sandbox` y `nosniff`). Con S3 se usan URLs prefirmadas. |

## Ajustes de tenant — `/tenant-settings` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/tenant-settings[/:scope[/:key]]` | Lista/lee settings; los secretos devuelven los campos no sensibles y redactan credenciales. |
| PUT | `/tenant-settings/:scope/:key` | Crea/actualiza (`{ value, isSecret }`); cifrado at-rest si es secreto. |
| DELETE | `/tenant-settings/:scope/:key` | Elimina el setting. |
| POST | `/tenant-settings/notifications/smtp/test` | Email de prueba con la config SMTP del tenant. |

## Auditoría — `/audit` (admin o auditor)

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/audit/entries` | admin/auditor | Log de auditoría con filtros (`actorId`, `action`, `resourceType`, fechas). En Community el rango se trunca a 90 días; con `feat:audit.long_retention` es ilimitado. |
| GET | `/audit/verify` | admin/auditor | Verifica la integridad de la cadena hash del log. |
| GET | `/audit/retention-info` | admin/auditor | Política activa: `{ plan, maxDays, capability }`. |
| GET | `/audit/entries.zip?from=&to=` | admin + `feat:reports.advanced_signed` | Export ZIP **firmado** (manifest + NDJSON + firma), verificable offline. **402** sin capability. |

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
| GET · POST | `/scim/v2/Users` | Lista con paginación SCIM y filtro `userName eq` · crea usuario (201). |
| GET · PATCH · DELETE | `/scim/v2/Users/:id` | Lee · aplica `PatchOp` (active, name, locale…) · soft-delete (204). |

Los cinco CRUD requieren la capability `feat:scim` (**402** sin licencia). El token se emite en `/admin/scim/token`.

## Inscripción externa — `/inscribe` (API keys)

| Método | Ruta | Scope | Qué hace |
|---|---|---|---|
| POST | `/inscribe` | `enrollments:write` | Crea-o-reusa usuario por email y lo matricula en `courseIds`/`accessGroupIds`. Idempotente. |
| POST | `/inscribe/revoke` | `enrollments:write` | Baja de matrículas de origen API (reembolso). Email inexistente → `userFound: false`, no 404. |
| GET | `/inscribe/courses` | `courses:read` | Catálogo con `status`, para mapear producto → curso. |
| GET | `/inscribe/access-groups` | `courses:read` | Grupos de acceso con `kind` y `courseCount`. |

Auth: `Authorization: ApiKey lmsk_…`. Un JWT de usuario no pasa estos endpoints: exigen scopes de API key.

## Webhooks salientes — `/webhooks` (admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/webhooks/info` | Tier activo, límites efectivos y catálogo de eventos suscribibles. |
| GET | `/webhooks/endpoints[/:id]` | Lista / detalle (secret enmascarado). |
| POST | `/webhooks/endpoints` | Crea: `{ url (https), eventTypes[] («*» = todos), secret?, active? }`. Secret en claro **una sola vez**. |
| PUT | `/webhooks/endpoints/:id` | Actualiza; enviar `secret` lo rota (one-shot). |
| DELETE | `/webhooks/endpoints/:id` | Elimina (204, idempotente). |

Errores: 409 URL duplicada · 422 `webhook_limit_exceeded` al superar el límite del plan (Community: 1 endpoint / 3 tipos de evento; Enterprise: 20 / ilimitados).
