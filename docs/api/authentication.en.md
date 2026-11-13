# Authentication

The API uses **stateless JWT Bearer** tokens (no cookies, no server sessions). There are two ways to authenticate: as a user (session tokens) or as an integration (API keys).

## 1. User sessions (JWT)

```bash
# Sign in
curl -X POST https://your-instance/api/v1/auth/signin \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "..."}'
# → { "accessToken": "...", "refreshToken": "..." }

# Authenticated calls
curl -H "Authorization: Bearer <accessToken>" https://your-instance/api/v1/me/profile

# Refresh
curl -X POST https://your-instance/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "..."}'
```

Token details:

- Signed HS256 with `AUTH_SECRET`; claims: user (`sub`), `tenantId`, `roles[]`, `mfaVerified` and `sid` (the session id).
- The `sid` references a session row in the database that enables **immediate revocation**: closing a session from `/me/security/sessions` (or suspending the account) cuts access without waiting for the token to expire — an interceptor checks it on every request.
- `POST /auth/forgot-password` always returns 200 (to prevent user enumeration).

### MFA

Optional per-user TOTP (`/auth/mfa/setup` → `/auth/mfa/enable` → `/auth/mfa/verify`). It can be made mandatory for administrators instance-wide (`DIDACTA_REQUIRE_MFA_ADMIN`) or by organization policy (the `feat:mfa.enforcement` Enterprise capability). Where MFA is required and not verified, the API returns `403 {"code": "mfa_required"}`.

## 2. API keys (integrations)

For external systems. Each key belongs to a user and their organization, and carries explicit **scopes**:

```bash
# Create one (as an admin) — the plaintext token is returned only ONCE
curl -X POST https://your-instance/api/v1/auth/api-keys \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "CRM", "scopes": ["enrollments:write", "courses:read"], "expiresAt": null}'
# → { "token": "lmsk_..." }

# Use it — the ApiKey scheme, not Bearer
curl -H "Authorization: ApiKey lmsk_..." https://your-instance/api/v1/inscribe/courses
```

- Scopes available today: `enrollments:write` (creating/removing enrollments through `/inscribe`) and `courses:read` (catalog and access groups).
- Keys are listed without their token and revoked with `DELETE /auth/api-keys/:id`.
- Endpoints with scopes are effectively **API-key only**: an admin JWT carries no scopes and does not pass the guard.

## 3. Corporate SSO

- **OIDC** (`/auth/oidc/*`) and **SAML 2.0** (`/auth/saml/*`) — *configuring* them is Enterprise (the `feat:sso.oidc` / `feat:sso.saml` capabilities); when unconfigured, the public flow returns 404.
- **WP-SSO** (`/modules/wp-sso/*`) — entry from a WordPress session with a short HMAC token (a Community module).
- **SCIM 2.0** (`/scim/v2/*`) — provisioning from Okta/Entra with its own static bearer token, issued in the panel (Enterprise).

## Real-time notifications (SSE)

`EventSource` does not accept headers, so the notification stream uses an **ephemeral ticket**: `POST /me/notifications/stream-ticket` returns a 60-second token that is passed as a query parameter when connecting to the stream.
