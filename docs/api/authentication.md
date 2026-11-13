# Autenticación

La API usa **JWT Bearer stateless** (sin cookies ni sesiones de servidor). Hay dos formas de autenticarse: como usuario (tokens de sesión) o como integración (API keys).

## 1. Sesiones de usuario (JWT)

```bash
# Login
curl -X POST https://tu-instancia/api/v1/auth/signin \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "..."}'
# → { "accessToken": "...", "refreshToken": "..." }

# Llamadas autenticadas
curl -H "Authorization: Bearer <accessToken>" https://tu-instancia/api/v1/me/profile

# Renovar
curl -X POST https://tu-instancia/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "..."}'
```

Detalles del token:

- Firmado HS256 con `AUTH_SECRET`; claims: usuario (`sub`), `tenantId`, `roles[]`, `mfaVerified` y `sid` (id de sesión).
- El `sid` referencia una fila de sesión en BD que permite **revocación inmediata**: cerrar una sesión desde `/me/security/sessions` (o suspender la cuenta) corta el acceso sin esperar a que caduque el token — un interceptor lo comprueba en cada petición.
- `POST /auth/forgot-password` responde 200 siempre (anti-enumeración de usuarios).

### MFA

TOTP opcional por usuario (`/auth/mfa/setup` → `/auth/mfa/enable` → `/auth/mfa/verify`). Puede hacerse obligatorio para administradores a nivel de instancia (`DIDACTA_REQUIRE_MFA_ADMIN`) o por política de organización (capability Enterprise `feat:mfa.enforcement`). Sin MFA verificado donde se exige, la API responde `403 {"code": "mfa_required"}`.

## 2. API keys (integraciones)

Para sistemas externos. Cada key pertenece a un usuario y su organización, y lleva **scopes** explícitos:

```bash
# Crear (como admin) — el token en claro solo se devuelve UNA vez
curl -X POST https://tu-instancia/api/v1/auth/api-keys \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "CRM", "scopes": ["enrollments:write", "courses:read"], "expiresAt": null}'
# → { "token": "lmsk_..." }

# Usar — esquema ApiKey, no Bearer
curl -H "Authorization: ApiKey lmsk_..." https://tu-instancia/api/v1/inscribe/courses
```

- Scopes disponibles hoy: `enrollments:write` (alta/baja de matrículas vía `/inscribe`) y `courses:read` (catálogo y grupos de acceso).
- Las keys se listan sin token y se revocan con `DELETE /auth/api-keys/:id`.
- Los endpoints con scopes son efectivamente **exclusivos de API keys**: un JWT de admin no porta scopes y no pasa el guard.

## 3. SSO corporativo

- **OIDC** (`/auth/oidc/*`) y **SAML 2.0** (`/auth/saml/*`) — la *configuración* es Enterprise (capabilities `feat:sso.oidc` / `feat:sso.saml`); sin configurar, el flujo público responde 404.
- **WP-SSO** (`/modules/wp-sso/*`) — entrada desde una sesión WordPress con token corto HMAC (módulo Community).
- **SCIM 2.0** (`/scim/v2/*`) — aprovisionamiento desde Okta/Entra con Bearer estático propio, emitido en el panel (Enterprise).

## Notificaciones en tiempo real (SSE)

`EventSource` no admite headers, así que el stream de notificaciones usa un **ticket efímero**: `POST /me/notifications/stream-ticket` devuelve un token de 60 segundos que se pasa como query param al conectar el stream.
