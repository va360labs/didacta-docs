# The Didacta API

All of Didacta's functionality is exposed through a **REST API** (NestJS + Fastify). The official web app is just another client: anything the panel does can be done over the API.

## The essentials

| | |
| --- | --- |
| **Base URL** | `https://your-instance/api/v1` |
| **Format** | JSON (16 MB body limit; 50 MB for uploading module ZIPs) |
| **Authentication** | JWT Bearer (user sessions), **API keys** (`ApiKey lmsk_…`), the provisioning credential (`Provisioning didprov_…`) and agent keys for MCP — see [Authentication](authentication.md) |
| **Interactive documentation** | Swagger UI at `/api/docs` · OpenAPI JSON at `/api/docs.json` |
| **Versioning** | `v1` in the path; no header negotiation |

Outside the `/api/v1` prefix sit the operational routes: `/healthz`, `/readyz`, `/livez` (probes), `/metrics` (Prometheus), `/api/docs` (Swagger), `/api/license` (public license status) and `/scim/v2/*` (SCIM).

## Your first request

The most direct way to integrate is with an **API key** (an administrator creates it in the panel or through `POST /api/v1/auth/api-keys`):

```bash
# Course catalog visible to integrators
curl -H "Authorization: ApiKey lmsk_xxx" \
  https://your-instance/api/v1/inscribe/courses
```

The enrollment API (`/inscribe`) is the surface meant for external integrators: create a student by email and enroll them in courses or access groups **idempotently**:

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

## How the surface is organized

| Block | Prefix | What it covers |
| --- | --- | --- |
| Authentication and identity | `/auth`, `/me`, `/users` | Sign-in, tokens, MFA, API keys, profile, sessions, support access. |
| Setup | `/setup` | First start of the instance (closes after initialization). |
| Public classroom site | `/public/*`, `/tenancy/*`, `/purchase-consent/*` | Public home, SEO, who is selling, domain and language resolution. No session. |
| Classroom administration | `/admin/*` | Users, invitations, modules, SMTP, Stripe, SSO, SCIM, domains, webhooks, AI, simple mode, sales, video, moderation. |
| Instance and control plane | `/admin/tenants`, `/admin/provisioning-credential`, `/admin/license`, `/admin/instance-settings` | Organizations, the provisioning credential, support access, the subscription gate, the license. See [Instance and control plane](reference/instance-operations.md). |
| Functional modules | `/modules/<slug>/*` | Courses, learning, community, assessments, certificates, payments, payouts, retention, support, Zoom, Fundae… |
| External integration | `/inscribe`, `/integrations`, `/community-api` | Idempotent enrollment, reading course pages and purchases, publishing to the community. |
| AI agents | `/agent/keys`, `/mcp` | Agent keys and the classroom's MCP server — see [MCP](mcp.md). |
| Outbound webhooks | `/webhooks`, `/admin/webhooks` | Endpoints subscribed to domain events — see [Webhooks](webhooks.md). |
| Platform | `/branding`, `/storage`, `/audit`, `/tenant-settings`, `/video` | Branding, files, auditing, tenant settings, video. |

The [endpoint reference](reference/index.md) walks through every block; for exhaustive per-endpoint detail (parameters, schemas, responses), use the **Swagger of your own instance** (`/api/docs`) — it always reflects the exact version you have deployed.

!!! note "Edition marks"
    In the reference, <span class="didacta-chip didacta-chip--ee">EE</span> means the endpoint requires an Enterprise capability (without a license it answers **402**). **Cloud** means something that only exists when the operator turns it on —the payouts platform mode or the video upload of Didacta Cloud—; on a self-hosted installation without that configuration those routes do not exist.

## Next step

- [Authentication](authentication.md) — JWT, API keys and scopes, the provisioning credential, SSO.
- [Conventions](conventions.md) — multi-tenancy, errors, rate limiting, pagination.
- [Endpoint reference](reference/index.md) — the complete map of the surface.
