# API conventions

## Multi-tenancy

Didacta is multi-tenant; every request operates in the context of an organization:

- **In authenticated calls the tenant comes from the credential** (the JWT's `tenantId` claim, or the API key's tenant) — not from the domain you call.
- The request **host** only comes into play where there is no credential yet: `GET /auth/tenant-context`, `signup`/`signin`/`forgot-password`, the initial `setup` and the classroom's public endpoints (`/public/*`, `/tenancy/*`, `/purchase-consent/*`, `/modules/billing/public/*`, `/membership/*`, `/modules/referrals/track`…). `X-Forwarded-Host` is read and, if absent, `Host`. Resolution requires an **exact** match against a verified domain (no subdomain wildcards). A domain that does not belong to any active classroom answers `404` on the `/public/*` endpoints and `tenant: null` on `GET /auth/tenant-context`.
- Calls with the **provisioning credential** have no tenant: they operate on the instance's list of organizations.
- In the database, isolation is reinforced with Row-Level Security on `tenant_id`.

### Classroom states that change the responses

| State | Effect on the API |
| --- | --- |
| Organization `SUSPENDED` or `ARCHIVED` | Its sessions stop being valid (401) and its domain stops resolving; `GET /tenancy/resolve` flags it with `suspended: true`. |
| Subscription gate `LOCKED` (control plane) | **402** `SUBSCRIPTION_LOCKED` on almost the whole API. Still answering: sign-in (`/auth/*`), `/me/profile`, `/me/modules`, branding (`/modules/theming/me` and logos), probes, `/api/license`, incoming payment-gateway and Zoom webhooks, and the organization routes. |
| Sign-ups frozen | **409** `TENANT_SIGNUPS_FROZEN` on single invitation, bulk upload, public registration and new-student checkout. People already inside are not affected. |

## Language

- The API's error messages (`message`) are in Spanish; **always program against `code`**, which is stable. The panel translates them by `code` (es/en/pt).
- Each classroom has a **language** (`es`, `en` or `pt`; `GET/PUT /admin/tenant-settings/academy`) that sets the language of emails and of whatever the server writes to everyone. Each person also has a profile `locale` (`es-ES`, `es-AR`, `en-US`, `pt-BR`; `PATCH /me/profile`).
- `GET /tenancy/locale` returns the language of the classroom for the request's domain. Anonymous checkout reads the web's `didacta_locale` cookie to store the buyer's language on their account, and `POST /setup/init` uses `Accept-Language` when no language is given; beyond that, the API does not negotiate language by header.

## Errors

The standard shape is `{ "statusCode", "message", "code"? }` plus extra fields depending on the case. `code` is the field to program against.

| Situation | Code | Shape |
| --- | --- | --- |
| Input validation (Zod) | 400 | `{ "message": "…", "issues": [{ "path", "message", "code" }] }` — `message` joins the problems with " · " |
| Not authenticated / invalid token or key | 401 | `AUTH_TOKEN_INVALID`, `AUTH_API_KEY_INVALID`, `AUTH_MISSING_BEARER_HEADER`, `session_revoked`, `account_suspended`… |
| Enterprise capability not active | **402** | `{ "statusCode": 402, "error": "CapabilityRequiredError", "code": "DIDACTA_CAPABILITY_REQUIRED", "capability": "feat:…", "path", "timestamp" }` |
| Classroom paused (subscription gate) | **402** | `{ "code": "SUBSCRIPTION_LOCKED", "message": "…" }` |
| Module disabled for the tenant | 403 | `{ "code": "TENANT_MODULES_MODULE_NOT_ACTIVE", "detail": "<module>", … }` |
| Missing permissions | 403 | the resource's own `code` (`ADMIN_FORBIDDEN_ADMIN_ROLE_REQUIRED`, `AUTH_API_KEY_MISSING_SCOPES`…) · `mfa_required` if the second factor is missing |
| Sanctioned user | 403 | `{ "code": "user_restricted", "area", "reason", "expiresAt" }` — only on writes in the sanctioned area |
| Conflicts, already initialized, frozen sign-ups | 409 | standard shape |
| General rate limit | 429 | `{ "code": "rate_limit_exceeded", "scope", "tier", "limit", "retryAfterSeconds" }` + a `Retry-After` header |
| Sign-in rate limit | 429 | `{ "code": "AUTH_RATE_LIMITED", "retryAfterSeconds" }` |
| Domain errors | depends on the case | `{ "statusCode", "code", "message", … }` — each module maps its typed errors (stable `code`) |

## Rate limiting

There are **three buckets** with a 60-second window backed by Redis. Which one applies to you depends on how you arrive:

| Bucket | Who | What counts |
| --- | --- | --- |
| `user` | A person with a session (JWT) | Their requests, per user and classroom — they also count against the classroom cap. |
| `authenticated` | All of a classroom's traffic: people with a session, **API keys, agent keys and SCIM** | The cap for the whole classroom. Keys and SCIM only count here. |
| `public` | No credential | Per client IP. |

Every response includes:

```
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Tier
```

Default limits (requests per minute):

| Plan | `user` | `authenticated` (classroom) | `public` (per IP) |
| --- | --- | --- | --- |
| Community | 100 | 100 | 30 |
| Enterprise <span class="didacta-chip didacta-chip--ee">EE</span> | 1000 | 1000 | 300 |

- The plan is **per instance**: it comes from the `feat:api.rate_limit.elevated` capability. `GET /admin/rate-limit/info` (administrator) returns the active plan and the effective numbers, also without a license, so the upgrade can be shown.
- Tune them with `RATE_LIMIT_COMMUNITY_USER_PER_MIN`, `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN`, `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` (and the matching `RATE_LIMIT_ENTERPRISE_*`) and `RATE_LIMIT_WINDOW_SECONDS`. Without `RATE_LIMIT_COMMUNITY_USER_PER_MIN`, the per-user limit equals the classroom's; Enterprise values never fall below the Community ones.
- If Redis is unavailable the limiter **lets requests through** (it does not take the API down) and the headers show `Remaining` = `Limit`.
- Exempt: `/healthz`, `/readyz`, `/livez`, `/metrics`, `/api/docs` and `/api/license`.
- With limits of their own: [sign-in](authentication.md#sign-in-attempt-limit) (`AUTH_RATE_LIMITED`), messaging (20 messages/min and 30 "typing" signals/min per person, `MESSAGING_RATE_LIMITED`) and the MCP server (see [MCP](mcp.md)).

## Pagination

There is no single format; each family of endpoints uses its own:

| Where | Parameters | Response |
| --- | --- | --- |
| Administration lists (`/admin/users`, access groups, groups…) | `page`, `limit` (max 500 on users) | `{ items, total, page, limit, hasMore }` |
| Sales (`/admin/sales`, `/modules/payouts/sales`) | `page`, `pageSize` (max 100) | `{ items, page, pageSize, total, … }` with totals per currency |
| Messaging | `cursor` (50 per page, newest to oldest) | Messages + next cursor |
| Community feed | `limit` and filters | List |
| SCIM | `startIndex` (1-based) and `count` | SCIM `ListResponse` |

## Gating by edition and by module

- **Enterprise endpoint without a license** → `402` with the required capability in the body. The capability list is in [Enterprise → Capabilities](../enterprise/capabilities.md).
- **Endpoint of a module that is disabled** for the tenant → `403` `TENANT_MODULES_MODULE_NOT_ACTIVE` ("Module X is not active for this tenant"). It applies to everything under `/modules/<slug>/…`.
- **Routes that do not exist without operator configuration** → `404`: platform payouts (`/modules/payouts/*`, which need the platform mode) and video upload (`/video/uploads/*`, which needs Bunny Stream). Both are Didacta Cloud features; see the [reference](reference/index.md).

## Outgoing webhooks

Subscribe to domain events with your own HTTPS endpoints (**`/api/v1/webhooks/endpoints`**, admin role). Each delivery is a JSON `POST` with `event`, `data`, `occurredAt`, `tenantId`, `deliveryId` and the `learner` object (the person behind the event, or `null`). Headers: `X-Didacta-Event`, `X-Didacta-Tenant`, `X-Didacta-Attempt` and `X-Didacta-Delivery` — **deduplicate by `X-Didacta-Delivery`**. The endpoint's secret is returned **only once** when it is created.

- **Community**: 1 endpoint per organization, at most 3 event types (`["*"]` counts as one), direct synchronous delivery with a 5 s timeout and 1 retry, **unsigned**.
- **Enterprise** <span class="didacta-chip didacta-chip--ee">EE</span> (`feat:api.webhooks.high_throughput`): 20 endpoints, unlimited events, a queue with exponential backoff, **HMAC-SHA256 signature** (`X-Didacta-Signature`) and a dead-letter queue with retry from the panel.

The full event catalog, the exact format and signature verification are in [Webhooks](webhooks.md).

## Other conventions

- **Traceability**: you can send `x-trace-id`; if you don't, one is generated and propagated in the logs.
- **Security headers**: every response carries `X-Content-Type-Options: nosniff` and `Referrer-Policy: no-referrer`; the API's add a restrictive `Content-Security-Policy` and `X-Frame-Options: DENY`, and `Strict-Transport-Security` when it arrives over HTTPS (turn it off with `WEB_HSTS=off`). Uploaded files (`/storage/file/*`) carry their own `sandbox` CSP.
- **CORS is not enabled**: the API is meant to be consumed same-origin (the web uses rewrites) or server-to-server. A browser client from another origin would need a proxy of its own.
- **Idempotency** where it matters: `/inscribe`, `/integrations/orders` (by `source` + `reference`), Stripe webhooks, module activation and simple mode's 3-question setup (`quick-start`) are safe to retry.
- **Amounts**: always in the currency's smallest unit (`amountCents`) with the currency explicit; different currencies are never added together.
