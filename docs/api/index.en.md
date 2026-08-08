# The Didacta API

All of Didacta's functionality is exposed through a **REST API** (NestJS + Fastify). The official web app is just another client: anything the panel does can be done over the API.

## The essentials

| | |
| --- | --- |
| **Base URL** | `https://your-instance/api/v1` |
| **Format** | JSON (16 MB body limit; 50 MB for uploading module ZIPs) |
| **Authentication** | JWT Bearer (user sessions) or **API keys** (`ApiKey lmsk_…`) — see [Authentication](autenticacion.md) |
| **Interactive documentation** | Swagger UI at `/api/docs` · OpenAPI JSON at `/api/docs.json` |
| **Versioning** | `v1` in the path; no header negotiation |

Outside the `/api/v1` prefix sit the operational routes: `/healthz`, `/readyz`, `/livez` (probes), `/metrics` (Prometheus), `/api/docs` (Swagger), `/api/license` (license status) and `/scim/v2/*` (SCIM).

## Your first request

The most direct way to integrate is with an **API key** (created in the panel or through `POST /api/v1/auth/api-keys`):

```bash
# Course catalog visible to integrators
curl -H "Authorization: ApiKey lmsk_xxx" \
  https://your-instance/api/v1/inscribe/courses
```

The enrollment API (`/inscribe`) is the surface designed for external integrators: create a student by email and enroll them in courses or access groups **idempotently**:

```bash
curl -X POST https://your-instance/api/v1/inscribe \
  -H "Authorization: ApiKey lmsk_xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "student@example.com",
    "name": "Example Student",
    "courseIds": ["..."],
    "accessGroupIds": ["..."]
  }'
```

## How the surface is organised

| Block | Prefix | What it covers |
| --- | --- | --- |
| Authentication and identity | `/auth`, `/me`, `/users` | Sign-in, tokens, MFA, API keys, profile, sessions. |
| Setup | `/setup` | The instance's first start (it closes after initialisation). |
| Administration | `/admin/*` | Organizations, users, invitations, modules, SMTP, SSO, SCIM, domains, webhooks, AI. |
| Functional modules | `/modules/<slug>/*` | Courses, learning, community, assessments, certificates, payments, Zoom, Fundae… |
| External integration | `/inscribe` | Idempotent account creation and enrollment for external systems. |
| Outgoing webhooks | `/webhooks` | Managing endpoints subscribed to domain events. |
| Platform | `/branding`, `/storage`, `/audit`, `/tenant-settings` | Branding, files, auditing, tenant settings. |

The [endpoint reference](referencia/index.md) walks through every block; for exhaustive per-endpoint detail (parameters, schemas, responses), use **your own instance's Swagger** (`/api/docs`) — it always reflects the exact version you have deployed.

## Next step

- [Authentication](autenticacion.md) — JWT, API keys, scopes and SSO.
- [Conventions](convenciones.md) — multi-tenancy, errors, rate limiting.
- [Endpoint reference](referencia/index.md) — the full map of the surface.
