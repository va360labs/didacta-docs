# API conventions

## Multi-tenancy

Didacta is multi-tenant; every request operates in the context of an organization:

- **In authenticated calls the tenant comes from the token** (the JWT's `tenantId` claim, or the API key's tenant) — not from the domain you call.
- The request **host** only comes into play where there is no credential yet: `GET /auth/tenant-context` (which organization this domain maps to), `signup`/`signin`/`forgot-password` and the initial `setup`. Resolution requires an **exact** match against a verified domain (no subdomain wildcards).
- In the database, isolation is reinforced with Row-Level Security on `tenant_id`.

## Errors

| Situation | Code | Shape |
| --- | --- | --- |
| Input validation (Zod) | 400 | `{ "message": "…", "issues": [{ "path", "message", "code" }] }` |
| Not authenticated / invalid license | 401 | standard shape |
| Enterprise capability not active | **402** | `{ "statusCode": 402, "error": "CapabilityRequiredError", "capability": "feat:…", "path", "timestamp" }` |
| Module disabled for the tenant / permissions | 403 | standard shape |
| Already initialised, conflicts | 409 | standard shape |
| Rate limit | 429 | `{ "code": "rate_limit_exceeded", "tier", "limit", "retryAfterSeconds" }` + a `Retry-After` header |
| Domain errors | depends on the case | `{ "statusCode", "code", "message", … }` — each module maps its typed errors (`code` is stable so you can program against it) |

## Rate limiting

Global, per organization (or `anonymous`), with a 60-second window backed by Redis. Every response includes:

```
X-RateLimit-Limit
X-RateLimit-Remaining
X-RateLimit-Reset
X-RateLimit-Tier
```

Default limits: **100 req/min** authenticated and **30 req/min** anonymous on Community (configurable with `RATE_LIMIT_COMMUNITY_*`); the Enterprise tier (`feat:api.rate_limit.elevated`) raises them to 1000/300. Exempt: probes, `/metrics`, `/api/docs`, `/api/license`.

## Gating by edition and by module

- **An Enterprise endpoint with no license** → `402`, with the required capability in the body. The list of capabilities is in [Enterprise → Capabilities](../enterprise/capabilities.md).
- **An endpoint of a module disabled** for the tenant → `403` ("module X is not active for this tenant").

## Outgoing webhooks

Subscribe to domain events with your own HTTPS endpoints (**`/api/v1/webhooks/endpoints`**, admin role):

- Payload: a `POST` with `{ "event": "<type>", "data": {…} }` and an `X-Didacta-Event` header.
- Catalog: `*`, `learning.enrollment.created`, `learning.course.completed`, `learning.lesson.completed`, `community.post.created`, `community.comment.created`, `billing.subscription.created`, `billing.subscription.cancelled`, `fundae.group.closed`, `assessments.attempt.submitted` — with prefix subscriptions (`billing.*`).
- The endpoint's secret is returned **exactly once**, when you create it.
- **Community**: 1 endpoint per organization, up to 3 event types, direct delivery with 1 retry. **Enterprise** (`feat:api.webhooks.high_throughput`): 20 endpoints, unlimited events, a queue with exponential backoff, **HMAC-SHA256 signing** and a dead-letter queue with retry from the panel.

## Other conventions

- **Pagination** in administration listings: `{ items, total, page, limit, hasMore }`.
- **Traceability**: you can send `x-trace-id`; if you do not, one is generated and propagated through the logs.
- **CORS is not enabled**: the API is designed to be consumed same-origin (the web app uses rewrites) or server to server. A browser client from another origin would need a proxy of its own.
- **Idempotency where it matters**: `/inscribe`, the Stripe webhooks and module activation are all safe to retry.
