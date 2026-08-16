# Reference — Core and cross-cutting

Authentication, account, setup, license, branding, storage, auditing, notifications, SSO, SCIM, external enrollment and webhooks. Every route hangs off `/api/v1` except those marked with ⚠.

## Authentication — `/auth`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/auth/tenant-context` | Public | Resolves the tenant from the `Host` header and returns the sign-in screen branding (logo, headlines, stats), or `{ tenant: null }`. |
| POST | `/auth/signup` | Public | Registers a user in the tenant (if sign-up is enabled) and returns tokens. |
| POST | `/auth/signin` | Public | Email + password sign-in → tokens + `mfaRequired` + user data. |
| POST | `/auth/refresh` | Public (refresh token in the body) | Refreshes the access token and records IP/device on the session. |
| POST | `/auth/forgot-password` | Public | Sends the reset email. **Always returns 200** (anti-enumeration). |
| POST | `/auth/reset-password` | Public | Confirms the reset with the token from the email + a new password (12–128). |

Relevant errors: `signin` returns a generic 401 ("invalid credentials"); if the email exists in several organizations it returns `AMBIGUOUS_TENANT` with the candidate slugs so the sign-in can be repeated with `tenantSlug`. `signup` returns 403 if registration is disabled and 409 if the email already exists in the tenant.

## MFA — `/auth/mfa`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/auth/mfa/setup` | Bearer | Generates the TOTP secret + QR + recovery codes (it does not enable MFA yet). |
| POST | `/auth/mfa/enable` | Bearer | Confirms with the first 6-digit code; audits it and reissues tokens with `mfaVerified=true`. |
| POST | `/auth/mfa/verify` | Bearer | Verifies the second factor (TOTP or a recovery code) and elevates the session. |

## API keys — `/auth/api-keys`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/auth/api-keys` | Bearer | Creates an API key: `{ name, scopes[], expiresAt? }`. **The `lmsk_…` token is only ever returned here.** |
| GET | `/auth/api-keys` | Bearer | Lists the user's keys (without tokens). |
| DELETE | `/auth/api-keys/:id` | Bearer | Revokes it (sets `revokedAt`) → 204. |

## My account — `/me`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · PATCH | `/me/profile` | Bearer | Full profile · editing (name, bio, job title, locale, timezone, avatar, validated national ID). |
| GET · POST | `/me/onboarding/status` · `complete` | Bearer | Onboarding status (`missing[]`) · marking it complete (422 if fields are missing). |
| GET · PUT | `/me/notification-preferences` | Bearer | Category × channel matrix (`COMMUNITY\|LEARNING\|ASSESSMENTS\|SYSTEM` × `EMAIL\|IN_APP`). |
| POST | `/me/security/password` | Bearer | Changes the password after verifying the current one; **closes every session**. |
| GET | `/me/security/sessions` | Bearer | Up to 20 active sessions (dates, IP, user agent). |
| DELETE | `/me/security/sessions/:id` | Bearer | Closes one specific session (immediate effect). |
| POST | `/me/security/sessions/revoke-others` | Bearer | Closes every session. |
| GET | `/me/modules` | Bearer | The tenant's active modules + Enterprise capabilities (this feeds the menu). |

## Public profiles — `/users`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/users/public?ids=a,b,c` | Bearer | A batch of `{ id, name, avatarUrl }` (max. 100) for feed avatars. |
| GET | `/users/:id/public-profile` | Bearer | Public profile (name, avatar, job title, bio). Never email, national ID or roles. |

## Notifications — `/me/notifications`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/me/notifications` | Bearer | My in-app notifications (max. 100, most recent first). |
| POST | `/me/notifications/:id/read` · `read-all` | Bearer | Marks one / all as read (idempotent). |
| POST | `/me/notifications/stream-ticket` | Bearer | Issues a 60 s JWT ticket for the stream. |
| GET | `/me/notifications/stream?ticket=` | SSE ticket | **SSE** — real-time stream (`notification` / `ping`). |

## Setup — `/setup`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/setup/status` | Public | Does the instance already have an organization? |
| GET | `/setup/available-modules` | Public | Modules for the wizard (`isCore`, `enabledByDefault`). |
| POST | `/setup/init` | Public | First-start bootstrap. 409 `ALREADY_INITIALIZED` if tenants already exist. |

## Platform

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | ⚠ `/api/license` | Public | Public license status (status, capabilities, notices). Exempt from rate limiting. |
| GET | `/branding/options` | Public | The tenant's branding for the UI (`logoUrl`, `primaryColor`, `poweredByDidacta`). |
| GET · POST | `/branding/white-label/preview` · `configure` | admin + the `feat:white_label` capability | White-label status and configuration. Requires an administrator session; **402** without a license. |
| GET | `/system/version-check` | Public | A proxy to the Docker Hub tags for the "new version" banner (15 min cache). |
| GET | ⚠ `/healthz` · `/livez` | Public | Liveness (version, uptime). |
| GET | ⚠ `/readyz` | Public | Readiness: checks the database, Redis and storage; **503** if anything is degraded. |
| GET | ⚠ `/metrics` | `Bearer <METRICS_TOKEN>` if defined | Prometheus/OpenMetrics metrics. |

## Storage — `/storage`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/storage/upload` | Bearer (any role) | Uploads an image or document as base64 (max. 10 MiB, MIME from a closed list); optimises images to WebP. |
| POST | `/storage/optimize` | Bearer (instructor+) | Re-optimises an already uploaded image and returns the new URL. |
| GET | `/storage/file/*` | Public | Serves a file from local storage by its key (with `CSP: sandbox` and `nosniff`). With S3, presigned URLs are used. |

## Tenant settings — `/tenant-settings` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/tenant-settings[/:scope[/:key]]` | Lists/reads settings; secrets return their non-sensitive fields and redact credentials. |
| PUT | `/tenant-settings/:scope/:key` | Creates/updates (`{ value, isSecret }`); encrypted at rest if it is a secret. |
| DELETE | `/tenant-settings/:scope/:key` | Deletes the setting. |
| POST | `/tenant-settings/notifications/smtp/test` | Sends a test email using the tenant's SMTP configuration. |

## Auditing — `/audit` (admin or auditor)

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/audit/entries` | admin/auditor | The audit log with filters (`actorId`, `action`, `resourceType`, dates). On Community the range is truncated to 90 days; with `feat:audit.long_retention` it is unlimited. |
| GET | `/audit/verify` | admin/auditor | Verifies the integrity of the log's hash chain. |
| GET | `/audit/retention-info` | admin/auditor | The active policy: `{ plan, maxDays, capability }`. |
| GET | `/audit/entries.zip?from=&to=` | admin + `feat:reports.advanced_signed` | A **signed** ZIP export (manifest + NDJSON + signature), verifiable offline. **402** without the capability. |

## SSO (public flows)

| Method | Route | What it does |
|---|---|---|
| GET | `/auth/oidc/:tenantSlug/status` · `start` · `/auth/oidc/callback` | OIDC: is it enabled? · redirect to the IdP (state+nonce+PKCE) · the callback that issues the session and redirects to the frontend. |
| GET/POST | `/auth/saml/:tenantSlug/status` · `login` · `acs` · `metadata` | SAML 2.0: status · AuthnRequest · Assertion Consumer Service (form-urlencoded) · the SP's XML metadata. |
| GET | `/modules/wp-sso/:tenantSlug/status` · `callback?token=` | WP-SSO: public configuration · exchanging WordPress's HMAC token for a Didacta session (302). |

With no configuration enabled, the flows return 404. Errors always redirect to `/auth/error?reason=…` with readable codes (`state_expired`, `email_not_allowed`, `user_not_provisioned`…).

## SCIM 2.0 — ⚠ `/scim/v2` (its own SCIM bearer token)

| Method | Route | What it does |
|---|---|---|
| GET | `/scim/v2/ServiceProviderConfig` · `ResourceTypes` · `Schemas` | Discovery (not gated). |
| GET · POST | `/scim/v2/Users` | Lists with SCIM pagination and a `userName eq` filter · creates a user (201). |
| GET · PATCH · DELETE | `/scim/v2/Users/:id` | Reads · applies a `PatchOp` (active, name, locale…) · soft deletes (204). |

All five CRUD operations require the `feat:scim` capability (**402** without a license). The token is issued at `/admin/scim/token`. The full guide —attribute mapping, error format and what is **not** supported— is in [SCIM provisioning](../../enterprise/scim.md).

## External enrollment — `/inscribe` (API keys)

| Method | Route | Scope | What it does |
|---|---|---|---|
| POST | `/inscribe` | `enrollments:write` | Creates or reuses a user by email and enrolls them in `courseIds`/`accessGroupIds`. Idempotent. |
| POST | `/inscribe/revoke` | `enrollments:write` | Removes API-originated enrollments (a refund). A non-existent email → `userFound: false`, not a 404. |
| GET | `/inscribe/courses` | `courses:read` | The catalog with `status`, to map product → course. |
| GET | `/inscribe/access-groups` | `courses:read` | Access groups with `kind` and `courseCount`. |

Auth: `Authorization: ApiKey lmsk_…`. A user JWT does not pass these endpoints: they require API key scopes.

## Read access for external sites — `/integrations` (API keys)

The other half of `/inscribe`: that one writes, this one reads. Meant for an external site to render a course page with Didacta's data and know whether the visitor is already a student.

| Method | Route | Scope | What it does |
|---|---|---|---|
| GET | `/integrations/courses` | `courses:read` | Courses filterable by `slug`, by `status` or by `externalId` + `externalSource` (mapping from the source LMS). |
| GET | `/integrations/courses/:courseId` | `courses:read` | The full record: metadata, instructor, totals, **syllabus module by module** and the purchase offer. **Never** a lesson's `content`. |
| GET | `/integrations/learners/state` | `enrollments:read` | A student's state by `email`: enrollments, progress and `nextLesson` with a direct URL. `known:false` if the email belongs to nobody. |

`enrollments:read` is deliberately separate from `enrollments:write`: it allows asking about **any** email in the organisation.

Watch out for `hasAccess:false` + `status:"PAUSED"`: that is a **suspended** student (typically for non-payment) with intact progress, not somebody who never bought.

Full guide with examples: [Selling and integrating from outside](../integraciones-externas.md).

## Outgoing webhooks — `/webhooks` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/webhooks/info` | The active tier, the effective limits and the catalog of subscribable events. |
| GET | `/webhooks/endpoints[/:id]` | List / detail (with the secret masked). |
| POST | `/webhooks/endpoints` | Creates one: `{ url (https), eventTypes[] ("*" = all), secret?, active? }`. The plaintext secret is returned **exactly once**. |
| PUT | `/webhooks/endpoints/:id` | Updates it; sending `secret` rotates it (one-shot). |
| DELETE | `/webhooks/endpoints/:id` | Deletes it (204, idempotent). |

Errors: 409 for a duplicate URL · 422 `webhook_limit_exceeded` when the plan limit is exceeded (Community: 1 endpoint / 3 event types; Enterprise: 20 / unlimited).
