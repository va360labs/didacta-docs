# Reference — Core and cross-cutting

Authentication, account, setup, public license, branding, the classroom's public site, storage, auditing, notifications, SSO, SCIM, external integration, agents and webhooks. Every route hangs off `/api/v1` except those marked with ⚠.

## Authentication — `/auth`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/auth/tenant-context` | Public | Resolves the tenant from the `Host` header and returns the sign-in screens' branding (logo, favicon, cover, headlines, color, real figures, `membershipPageActive`, `subscriptionGate.state`: `LOCKED` only if the classroom is paused) or `{ tenant: null }`. |
| POST | `/auth/signup` | Public | Registers an `alumno` user in the tenant (if sign-up is enabled) and returns tokens. Accepts the browser's `timezone` and `locale` only to create the account. |
| POST | `/auth/signin` | Public | Email + password sign-in → `{ tokens, mfaRequired, user }` (plus `mfaRequiredSoon` while the MFA policy is in its grace period). |
| POST | `/auth/refresh` | Public (refresh token in the body) | Refreshes the tokens (`{ tokens }`) and records IP/device on the session. |
| POST | `/auth/forgot-password` | Public | Sends the reset email. **Always returns 200** (anti-enumeration). |
| POST | `/auth/reset-password` | Public | Confirms the reset with the token from the email + a new password (12–128). Returns `email` and `tenantSlug` so the person can sign in right away. |
| POST | `/auth/reset-password/status` | Public | State of an access link **without consuming it**: `valid`, `expired`, `used` or `invalid`. |
| POST | `/auth/reset-password/resend` | Public | With an **expired**, unused link, sends a new one to that account's email (once per link). Always answers 200. |

Relevant errors: `signin` returns 401 `AUTH_INVALID_CREDENTIALS` ("invalid credentials"); if the email exists in several organizations it returns `AMBIGUOUS_TENANT` with `candidateSlugs` so the sign-in can be repeated with `tenantSlug`. `signup` returns 403 `AUTH_SIGNUP_DISABLED` if registration is disabled and 409 `AUTH_EMAIL_TAKEN` if the email already exists in the tenant. These routes have their [own attempt limit](../authentication.md#sign-in-attempt-limit) (429 `AUTH_RATE_LIMITED`).

### Support access — `/auth/support-access`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/auth/support-access/redeem` | Public (single-use token) | Redeems the token the control plane opened for a time-limited session, **with no refresh token**. Errors: 400 `SUPPORT_ACCESS_INVALID`; 410 `SUPPORT_ACCESS_EXPIRED` / `_ALREADY_REDEEMED` / `_REVOKED`; 403 `SUPPORT_ACCESS_TENANT_UNAVAILABLE`. |
| GET | `/auth/support-access/current` | Bearer | `{ support: {…} }` if the session is a support one and still alive; `{ support: null }` on a normal session. The classroom uses it for its permanent notice. |

How the access is opened is in [Instance and control plane](instance-operations.md#support-access).

## MFA — `/auth/mfa`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/auth/mfa/setup` | Bearer | Generates the TOTP secret + QR + recovery codes (it does not enable MFA yet). |
| POST | `/auth/mfa/enable` | Bearer | Confirms with the first 6-digit code; audits it and reissues tokens with `mfaVerified=true`. |
| POST | `/auth/mfa/verify` | Bearer | Verifies the second factor (TOTP or a recovery code) and elevates the session. |

The organization-wide MFA policy is in [Administration](administration.md#corporate-identity-ee) (`/admin/mfa-policy`).

## API keys — `/auth/api-keys`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/auth/api-keys` | admin | Creates an API key: `{ name, scopes[], expiresAt? }`. Scopes come from a closed catalog (see [Authentication](../authentication.md#scopes)). **The `lmsk_…` token is only returned here.** |
| GET | `/auth/api-keys` | admin | Lists your own keys (no tokens). The whole classroom's are at `GET /admin/api-keys`. |
| DELETE | `/auth/api-keys/:id` | admin | Revokes (sets `revokedAt`) → 204. |

Only `tenant_admin` and `super_admin`: any other role gets 403 `ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`.

## My account — `/me`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · PATCH | `/me/profile` | Bearer | Full profile · edit (`name`, `bio` ≤280, `jobTitle`, `department`, `location`, `locale` —`es-ES`, `es-AR`, `en-US`, `pt-BR`—, `timezone`, `avatarUrl`, validated DNI/NIE). `''` or `null` clear a field; omitting it leaves it as is. |
| GET · POST | `/me/onboarding/status` · `complete` | Bearer | Onboarding state (`missing[]`) · mark it completed (422 `AUTH_ONBOARDING_MISSING_FIELDS` if name or avatar are missing). |
| GET · PUT | `/me/notification-preferences` | Bearer | Category × channel matrix (`COMMUNITY\|LEARNING\|ASSESSMENTS\|SYSTEM` × `EMAIL\|IN_APP`). With no row, the default is enabled. |
| POST | `/me/security/password` | Bearer | Changes the password verifying the current one; **closes all sessions**. |
| GET | `/me/security/sessions` | Bearer | Up to 20 active sessions (dates, IP, user agent). |
| DELETE | `/me/security/sessions/:id` | Bearer | Closes one specific session (immediate effect). |
| POST | `/me/security/sessions/revoke-others` | Bearer | Closes all sessions. |
| GET | `/me/modules` | Bearer | The tenant's active modules + Enterprise capabilities (feeds the menu); for administrators it also carries the subscription gate state. |
| GET | `/me/purchases` | Bearer | My purchases made in the center's external store (those that store left with `POST /integrations/orders`): amount and, if any, invoice number and a link to the PDF served by whoever issued it. Empty if no external store sells. Didacta does not generate invoices here. |

## Public profiles — `/users`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/users/public?ids=a,b,c` | Bearer | Batch of `{ id, name, avatarUrl }` (max 100) for feed avatars. |
| GET | `/users/:id/public-profile` | Bearer | Public profile (name, avatar, job title, department, location, bio). Never email, DNI or roles. |

## Notifications — `/me/notifications`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/me/notifications` | Bearer | My in-app notifications (max 100, newest first). |
| POST | `/me/notifications/:id/read` · `read-all` | Bearer | Marks one / all as read (idempotent). |
| POST | `/me/notifications/stream-ticket` | Bearer | Issues a 60 s JWT ticket for the stream. |
| GET | `/me/notifications/stream?ticket=` | SSE ticket | **SSE** — real-time stream (`notification` / `ping`). |

## Setup — `/setup`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/setup/status` | Public | Does the instance already have an organization? |
| GET | `/setup/available-modules` | Public | Modules for the wizard (`isCore`, `enabledByDefault`). |
| POST | `/setup/init` | Public | First-start bootstrap: organization (`name`, `slug?`, `primaryHostname?`, `locale?`), the first `super_admin` (`name`, `email`, `password`) and `enabledModules?`. Returns tokens. 409 `ALREADY_INITIALIZED` if tenants already exist. |

## Platform

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | ⚠ `/api/license` | Public | Public license state (status, capabilities, warnings). Exempt from rate limiting and from the subscription gate. License management (`/admin/license`) is in [Instance and control plane](instance-operations.md#license-adminlicense-super_admin). |
| GET | `/branding/options` | Public | The tenant's branding for the UI (`logoUrl`, `primaryColor`, `poweredByDidacta`). |
| GET · POST | `/branding/white-label/preview` · `configure` | super_admin <span class="didacta-chip didacta-chip--ee">EE</span> `feat:white_label` | White-label state and configuration. **402** without a license, 403 if you are not a super_admin. |
| GET | `/system/version-check` | Public | Proxy to the Docker Hub tags for the "new version" banner (15 min cache). |
| GET | ⚠ `/healthz` · `/livez` | Public | Liveness (version, uptime). |
| GET | ⚠ `/readyz` | Public | Readiness: checks DB, Redis and storage; **503** if something is degraded. |
| GET | ⚠ `/metrics` | `Bearer <METRICS_TOKEN>` | Prometheus/OpenMetrics metrics. With `METRICS_TOKEN` defined it requires that token; **in production without `METRICS_TOKEN` it answers 401**: define it in the API's environment. |

## The classroom's public site (no session)

Endpoints the web uses to render what a visitor sees. The classroom comes from the request's domain (`X-Forwarded-Host`, or `Host`); a domain that belongs to no active classroom answers 404.

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/tenancy/resolve` | Public | `{ known, slug, host, surface, suspended, redirectTo }`: which classroom the domain maps to and whether it serves the app (`APP`) or the public site (`SITE`). A suspended classroom's domain gives `known: false` and `suspended: true`. `redirectTo` is the primary domain when the host is a **previous address** of the classroom (the web app redirects its pages with 308; `/api/*` is not redirected); `null` in any other case. |
| GET | `/tenancy/locale` | Public | `{ locale: "es" \| "en" \| "pt" \| null }`: the classroom's language for the domain. |
| GET | `/public/academy-home` | Public | Public home: brand, member count, who runs it, next live class and access options (courses for sale, membership, free access). `{ enabled: false }` if the classroom has it turned off. |
| GET | `/public/seo` | Public | Indexing: whether it is indexed, canonical domain, language, public data and sales pages with their last modification. `{ indexable: false }` with the public page turned off. |
| GET | `/public/site-context` | Public | Tenant, active modules and canonical origin of the public site. 404 if the domain does not serve a public site. |
| GET | `/public/checkout-seller` | Public | Who sells in the classroom: name, who runs it and the configured contact email. |
| GET | `/purchase-consent/withdrawal-waiver` | Public | Text and version of the "I want to start now and lose the right of withdrawal" checkbox for this classroom, in the buyer's language. |

## Storage — `/storage`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/storage/upload` | student, instructor or admin | Uploads an image or document as base64 (max 10 MiB, closed MIME list); optimizes images to WebP. |
| POST | `/storage/optimize` | instructor+ | Re-optimizes an already uploaded image and returns the new URL. |
| GET | `/storage/file/*` | Public | Serves a local-storage file by its key (with `CSP: sandbox` and `nosniff`). With S3, presigned URLs are used. |

## Tenant settings — `/tenant-settings` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/tenant-settings[/:scope[/:key]]` | Lists/reads settings; secrets return the non-sensitive fields and redact credentials. |
| PUT | `/tenant-settings/:scope/:key` | Creates/updates (`{ value, isSecret }`); encrypted at rest if secret. |
| DELETE | `/tenant-settings/:scope/:key` | Deletes the setting. |
| POST | `/tenant-settings/notifications/smtp/test` | Test email using the tenant's SMTP config. |

Settings with their own screen (SMTP, Stripe, classroom) have dedicated endpoints in [Administration](administration.md).

## Auditing — `/audit` (admin or auditor)

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/audit/entries` | admin/auditor | Audit log with filters (`actorId`, `action`, `resourceType`, dates). On Community the range is truncated to 90 days; with `feat:audit.long_retention` it is unlimited. |
| GET | `/audit/verify` | admin/auditor | Verifies the integrity of the log's hash chain. |
| GET | `/audit/retention-info` | admin/auditor | Active policy: `{ plan, maxDays, capability }`. |
| GET | `/audit/entries.zip?from=&to=` | admin <span class="didacta-chip didacta-chip--ee">EE</span> `feat:reports.advanced_signed` | **Signed** ZIP export (manifest + NDJSON + signature), verifiable offline. **402** without the capability. |

## SSO (public flows)

| Method | Route | What it does |
|---|---|---|
| GET | `/auth/oidc/:tenantSlug/status` · `start` · `/auth/oidc/callback` | OIDC: enabled? · redirect to the IdP (state+nonce+PKCE) · callback that issues the session and redirects to the frontend. |
| GET/POST | `/auth/saml/:tenantSlug/status` · `login` · `acs` · `metadata` | SAML 2.0: status · AuthnRequest · Assertion Consumer Service (form-urlencoded) · the SP's XML metadata. |
| GET | `/modules/wp-sso/:tenantSlug/status` · `callback?token=` | WP-SSO: public config · exchange of WordPress's HMAC token for a Didacta session (302). |

Without an enabled configuration the flows answer 404. Errors always redirect to `/auth/error?reason=…` with readable codes (`state_expired`, `email_not_allowed`, `user_not_provisioned`…).

## SCIM 2.0 — ⚠ `/scim/v2` (its own SCIM Bearer)

| Method | Route | What it does |
|---|---|---|
| GET | `/scim/v2/ServiceProviderConfig` · `ResourceTypes` · `Schemas` | Discovery (not gated). |
| GET · POST | `/scim/v2/Users` | List with SCIM pagination and the `userName eq` filter · creates a user (201; 409 `uniqueness` if the `userName` already exists). |
| GET · PATCH · DELETE | `/scim/v2/Users/:id` | Reads · applies `PatchOp` (active, name, locale…) · soft-delete (204). |
| ANY | `/scim/v2/Groups[/*]` | **501** with a SCIM error body: Groups is not implemented. |

The five user CRUD routes require the `feat:scim` capability (**402** without a license). The token is issued at `/admin/scim/token`. Full guide —attribute mapping, error format and what is **not** supported— in [SCIM provisioning](../../enterprise/scim.md).

## External enrollment — `/inscribe` (API keys)

| Method | Route | Scope | What it does |
|---|---|---|---|
| POST | `/inscribe` | `enrollments:write` | Creates-or-reuses a user by email (temporary password sent by email) and enrolls them in `courseIds`/`accessGroupIds` (at least one). Accepts `name`, `locale` and `externalRef`. Idempotent. |
| POST | `/inscribe/revoke` | `enrollments:write` | Unenrolls API-origin enrollments (refund). Unknown email → `userFound: false`, not 404. It does not revoke access via group, subscription or direct purchase. |
| GET | `/inscribe/courses` | `courses:read` | Catalog with `status`, to map product → course. |
| GET | `/inscribe/access-groups` | `courses:read` | Access groups with `kind` and `courseCount`. |

Auth: `Authorization: ApiKey lmsk_…`. A user JWT does not pass these endpoints: they require API key scopes.

## Reading course pages and purchases for external sites — `/integrations` (API keys)

The other half of `/inscribe`: that one writes enrollments, this one reads course pages and stores purchase history. Meant so that an outside site can render a course page with Didacta data, know whether the viewer is already a student, and leave their orders in their profile.

| Method | Route | Scope | What it does |
|---|---|---|---|
| GET | `/integrations/courses` | `courses:read` | Courses filterable by `slug`, by `status` or by `externalId` + `externalSource` (mapping from the origin LMS). |
| GET | `/integrations/courses/:courseId` | `courses:read` | Full page: metadata, instructor, totals, **syllabus module by module** and purchase offer. **Never** a lesson's `content`. |
| GET | `/integrations/learners/state` | `enrollments:read` | A student's state by `email` (and `courseId?`): enrollments, progress, `nextLesson` with a direct URL and the live `membership` (or `null`). `known:false` if the email belongs to nobody. |
| POST | `/integrations/orders` | `orders:write` | Stores in the student's profile a purchase made in an external store. **Idempotent by `(source, reference)`**; `invoice`, `orderUrl` and `refundedAt` are only written if sent (omitting them does not delete; `invoice: null` withdraws the invoice). Didacta neither issues nor numbers invoices. |
| GET | `/integrations/learners/orders` | `orders:read` | A student's purchase history by `email`. `known: false` does not imply an empty list. |

`enrollments:read` is kept apart from `enrollments:write` on purpose: it lets you ask about **any** email in the organization.

Mind `hasAccess:false` + `status:"PAUSED"`: it is a **suspended** student (typically for non-payment) with their progress intact, not someone who never bought.

Full guide with examples: [Selling and integrating from outside](../external-integrations.md).

## AI agents — `/agent` and `/mcp`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/agent/keys` | admin | Creates an AI agent's key. The server fixes the scopes (`agent:courses:read`, `agent:courses:write`, `agent:live:write`, `agent:community:post`); the client does not choose them. **The token is only returned here.** |
| POST | `/mcp` | Agent key (`Bearer lmsk_…` or `ApiKey lmsk_…`) | The classroom's MCP server (Streamable HTTP, JSON-RPC 2.0). `GET` and `DELETE` on `/mcp` answer 405 (no stream, no sessions). |

Tools, per-key limits and what an agent **cannot** do: [MCP](../mcp.md).

## Outbound webhooks — `/webhooks` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/webhooks/info` | Active tier, effective limits and catalog of subscribable events. |
| GET | `/webhooks/endpoints[/:id]` | List / detail (masked secret). |
| POST | `/webhooks/endpoints` | Creates: `{ url (https), eventTypes[] ("*" = all), secret?, active? }`. Plain secret **only once**. |
| PUT | `/webhooks/endpoints/:id` | Updates; sending `secret` rotates it (one-shot). |
| DELETE | `/webhooks/endpoints/:id` | Deletes (204, idempotent). |

Errors: 409 duplicate URL · 422 `webhook_limit_exceeded` when the plan's limit is exceeded (Community: 1 endpoint / 3 event types; Enterprise: 20 / unlimited). The failed-retry queue (dead-letter, EE) is in [Administration](administration.md#corporate-identity-ee). The delivery format, the event catalog and the signature are in [Webhooks](../webhooks.md).
