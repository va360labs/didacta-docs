# Enterprise capabilities

The **closed** list of Enterprise capabilities — 11 today, defined in `packages/license-sdk/src/capabilities.ts`. An active license carries the purchased capabilities in its payload; everything else is Community.

## Identity and access

| Capability | What it unlocks |
| --- | --- |
| `feat:sso.saml` | Configuration of **SAML 2.0** corporate sign-in (`/admin/sso/saml`). |
| `feat:sso.oidc` | Configuration of **OIDC** corporate sign-in (`/admin/sso/oidc`). |
| `feat:scim` | **SCIM 2.0 provisioning** from Okta/Entra: automatic user creation, deactivation and updates (`/scim/v2` + token issuing). |
| `feat:mfa.enforcement` | **Mandatory MFA at the organization level** (a policy enforced at sign-in). Community allows optional per-user MFA. |

## Platform

| Capability | What it unlocks |
| --- | --- |
| `feat:multi_tenant.real` | **Real multi-tenancy**: creating multiple isolated organizations on the same instance. Community operates with one organization. |
| `feat:custom_domains` | **Per-tenant custom domains** with CNAME verification managed from the panel. |
| `feat:white_label` | **Full white-label**: overriding the brand and hiding "powered by Didacta". Basic branding (logo, colors, copy) is Community. |

## Compliance

| Capability | What it unlocks |
| --- | --- |
| `feat:audit.long_retention` | **Signed 7-year** retention of the audit log. Community: 90 days. |
| `feat:reports.advanced_signed` | **Cryptographically signed XLSX/PDF exports** (verifiable offline). |

## API

| Capability | Community | With the capability |
| --- | --- | --- |
| `feat:api.webhooks.high_throughput` | 1 endpoint per organization, up to 3 event types, direct delivery with 1 retry | 20 endpoints, unlimited events, a queue with backoff, **HMAC-SHA256 signing**, dead-letter with retry |
| `feat:api.rate_limit.elevated` | 100 req/min authenticated · 30 anonymous | 1000 req/min authenticated · 300 anonymous |

## How gating is applied

- **Backend**: Enterprise endpoints carry `@RequiresCapability(...)` (or `license.requireCapability(...)` in the service) and return **402** with the required capability when it is not active:

    ```json
    {
      "statusCode": 402,
      "error": "CapabilityRequiredError",
      "capability": "feat:sso.saml",
      "path": "/api/v1/admin/sso/saml/config"
    }
    ```

- **Frontend**: the screen always exists; the real panel sits inside a gate that, without the capability, shows the upsell notice. The web app reads the active capabilities from `GET /api/license` and `GET /api/v1/me/modules`.
- **Code**: only the capabilities whose code is physically separated live in `*.ee.*` files inside the core (covered by the Enterprise license); the rest are gated inline. No `.ee` file may exist under `modules/` — the `ee-fence` validator guarantees this in CI.
