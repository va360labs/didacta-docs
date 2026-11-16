# Authentication

The API uses **stateless credentials in the `Authorization` header**: it sets no session cookies and keeps no server-side state to authenticate you. Depending on who is calling there are four different credentials:

| Who | Credential | Header | Where it works |
| --- | --- | --- | --- |
| A person (panel, app, scripts) | Session JWT | `Authorization: Bearer <accessToken>` | Almost the whole API, depending on their role |
| An external system (store, CRM, automation) | API key `lmsk_…` | `Authorization: ApiKey lmsk_…` | Only `/inscribe`, `/integrations` and `/community-api`, depending on its scopes |
| An AI agent | Agent key (an API key with `agent:*` scopes) | `Authorization: Bearer lmsk_…` or `ApiKey lmsk_…` | Only the MCP server (`/mcp`) — see [MCP](mcp.md) |
| The instance's control plane (Didacta Cloud or your own script) | Provisioning credential `didprov_…` | `Authorization: Provisioning didprov_…` | A whitelist of routes over organizations — see [below](#3-provisioning-credential) |

## 1. User sessions (JWT)

```bash
# Sign in
curl -X POST https://your-instance/api/v1/auth/signin \
  -H "Content-Type: application/json" \
  -d '{"email": "admin@example.com", "password": "..."}'
# → {
#     "tokens": { "accessToken": "...", "refreshToken": "...", "expiresIn": 3600 },
#     "mfaRequired": false,
#     "user": { "id": "…", "email": "…", "name": "…", "avatarUrl": null,
#               "tenantId": "…", "tenantSlug": "demo" }
#   }

# Authenticated calls
curl -H "Authorization: Bearer <accessToken>" https://your-instance/api/v1/me/profile

# Refresh → { "tokens": { … } }
curl -X POST https://your-instance/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{"refreshToken": "..."}'
```

Token details:

- Signed HS256 with `AUTH_SECRET`. The **access token lasts 1 hour** and the **refresh token 30 days**. Claims: user (`sub`), `tenantId`, `roles[]`, `mfaVerified` and `sid` (session id).
- **Immediate revocation.** The `sid` references a session row in the database. Closing a session from `/me/security/sessions`, suspending the account or suspending the classroom cuts access without waiting for the token to expire: every request checks the account state and the API answers **401** with `code` `session_revoked`, `account_suspended` or `account_deleted` (`/auth/*` routes are left out so a suspended account can still sign out).
- `POST /auth/forgot-password` always answers 200 (user anti-enumeration).
- **You don't have to say which organization you are signing into.** Sign-in resolves the classroom from the request domain; `tenantSlug` is optional and only needed when the domain is not mapped. With neither a domain nor a slug, the email is looked up across active organizations: if it exists in more than one, the answer is `AMBIGUOUS_TENANT` with the candidate slugs.

### Sign-in attempt limit

`signin`, `signup`, `forgot-password`, `reset-password`, link resend, MFA verification and support-access redemption have a **limiter of their own, stricter than the general one**: by default 30 attempts per minute per IP, and 10 failures in 15 minutes per account. When exceeded it answers **429** with `code: "AUTH_RATE_LIMITED"` and `retryAfterSeconds`; the body is identical whether or not the account exists. Tune it with `RATE_LIMIT_LOGIN_IP_MAX`, `RATE_LIMIT_LOGIN_IP_WINDOW_SECONDS`, `RATE_LIMIT_LOGIN_ACCOUNT_MAX` and `RATE_LIMIT_LOGIN_ACCOUNT_WINDOW_SECONDS`. If Redis is unavailable the limiter fails open (lets requests through) rather than taking sign-in down.

### MFA

Optional TOTP per user (`/auth/mfa/setup` → `/auth/mfa/enable` → `/auth/mfa/verify`). It can be made mandatory for administrators at instance level (`DIDACTA_REQUIRE_MFA_ADMIN`) or through an organization policy (<span class="didacta-chip didacta-chip--ee">EE</span> capability `feat:mfa.enforcement`). With the requirement on and no verified MFA, the API answers `403 {"code": "mfa_required"}` on everything except the MFA flow itself and the minimal profile.

### Roles

The `roles[]` claim carries the person's roles: `super_admin`, `tenant_admin`, `formador` (instructor), `alumno` (student), `auditor`, `empresa_manager` and `inspector` (read-only Fundae follow-up, limited to the groups they are granted). `super_admin` belongs to the instance and **cannot be assigned through the API**; the others are assigned by an administrator (`POST /admin/users/:id/roles`).

### Support access

The control plane can open a support access window into a classroom (see [Instance and control plane](reference/instance-operations.md#support-access)). It is redeemed with `POST /auth/support-access/redeem` and yields an `accessToken` **with no refresh token** that expires with the grant (15 minutes at most). The token carries the `sup` claim; the session belongs to a support user of the classroom's own, visible in its member list, and it cannot, for example, export personal sales data.

## 2. API keys (integrations)

For external systems. Each key belongs to an administrator user of an organization and carries explicit **scopes**. **Only administrators (`tenant_admin`, `super_admin`) create and manage keys**: a key grants power over the whole classroom (enrolling anyone, reading other people's purchases), not over the person who creates it. An instructor (`formador`) or a student (`alumno`) gets `403 ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`.

```bash
# Create (as admin) — the plain token is returned ONCE
curl -X POST https://your-instance/api/v1/auth/api-keys \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{"name": "CRM", "scopes": ["enrollments:write", "courses:read"], "expiresAt": "2027-12-31T00:00:00Z"}'
# → { "token": "lmsk_...", … }

# Use — ApiKey scheme, not Bearer
curl -H "Authorization: ApiKey lmsk_..." https://your-instance/api/v1/inscribe/courses
```

### Scopes

Scopes come from a **closed catalog**: asking for one that is not on the list gives `400`, and an old key holding a scope outside the catalog loses it when used. `expiresAt` is optional (ISO-8601); without it the key does not expire until revoked.

| Scope | What it allows |
| --- | --- |
| `enrollments:write` | `POST /inscribe` and `POST /inscribe/revoke`: enrolling and unenrolling. |
| `enrollments:read` | `GET /integrations/learners/state`: the state of **any** email in the classroom. Kept apart from `write` on purpose. |
| `courses:read` | `GET /inscribe/courses`, `GET /inscribe/access-groups` and `GET /integrations/courses[/:id]`: catalog and course pages. |
| `orders:write` | `POST /integrations/orders`: store in a student's profile a purchase made in an external store. |
| `orders:read` | `GET /integrations/learners/orders`: a student's purchase history by email. |
| `community:post` | `/community-api/*`: list spaces and publish posts (the key's owner must be an administrator). |
| `agent:courses:read`, `agent:courses:write`, `agent:live:write`, `agent:community:post` | The agent gate (`/mcp`). Created with `POST /agent/keys` —the server fixes the four scopes—; see [MCP](mcp.md). |

- Keys are listed without the token (`GET /auth/api-keys`, or `GET /admin/api-keys` for those of the **whole classroom**) and revoked with `DELETE` (`revokedAt` is set; the row is not deleted).
- Endpoints with scopes are effectively **API-key-only**: an admin JWT carries no scopes and does not pass the guard.
- API keys **do not go through the per-user quota** of the rate limit: they count against the organization bucket (see [Conventions](conventions.md#rate-limiting)).
- Every creation and revocation is recorded in the audit log (`admin.api_key.created` and `admin.api_key.revoked`).

## 3. Provisioning credential

An **instance** credential, not an organization one: an external control plane (Didacta Cloud, or a script of yours) uses it to operate the instance's list of organizations without impersonating a person. It has no tenant and no user, and it only opens the routes flagged as machine-operable (list, view, create and change the status of organizations, freeze sign-ups, the subscription gate, support access, domains, usage, video and platform payouts; plus `GET /admin/system/health-detail`). The detail is in [Instance and control plane](reference/instance-operations.md).

```bash
# A super_admin issues it. The plain token is returned ONCE;
# issuing another revokes the previous one (at most one is alive per instance).
curl -X POST https://your-instance/api/v1/admin/provisioning-credential \
  -H "Authorization: Bearer <super_admin accessToken>"
# → { "id": "…", "token": "didprov_…", "preview": "didprov_…", "createdAt": "…" }

# Use it
curl -H "Authorization: Provisioning didprov_..." https://your-instance/api/v1/admin/tenants

# Revoke it (idempotent)
curl -X DELETE https://your-instance/api/v1/admin/provisioning-credential \
  -H "Authorization: Bearer <super_admin accessToken>"
```

- Only the SHA-256 of the credential is stored (encrypted at rest): if you lose the token, you issue another.
- **401** `AUTH_PROVISIONING_CREDENTIAL_INVALID` if it is invalid or revoked; **403** `AUTH_PROVISIONING_ROUTE_NOT_ALLOWED` if it is valid but the route is not on the whitelist (for example, renaming an organization requires a super_admin with a session).
- On a single-process installation, revoking cuts access on the next request; with several replicas, up to 60 seconds later on the others.
- What it does is recorded in the audit log with `actorKind: "provisioning"` and the credential id, distinguishable from a human action.

## 4. AI agents (MCP)

The classroom exposes an MCP server at `POST /api/v1/mcp` (Streamable HTTP, JSON-RPC 2.0) using agent keys. How to connect a client, which tools exist and what an agent **cannot** do is on the [MCP](mcp.md) page.

## 5. Corporate SSO

- **OIDC** (`/auth/oidc/*`) and **SAML 2.0** (`/auth/saml/*`) — *configuring* them is <span class="didacta-chip didacta-chip--ee">EE</span> (capabilities `feat:sso.oidc` / `feat:sso.saml`); when not configured, the public flow answers 404.
- **WP-SSO** (`/modules/wp-sso/*`) — sign-in from a WordPress session with a short HMAC token (Community module).
- **SCIM 2.0** (`/scim/v2/*`) — provisioning from Okta/Entra with its own static Bearer, issued in the panel (<span class="didacta-chip didacta-chip--ee">EE</span> `feat:scim`).

## Real-time notifications (SSE)

`EventSource` does not support headers, so the streams use an **ephemeral ticket**: `POST /me/notifications/stream-ticket` (and `POST /modules/messaging/stream-ticket` for messaging) returns a 60-second token that is passed as a query param (`?ticket=`) when connecting the stream.
