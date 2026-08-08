# The Enterprise license

Didacta's license is a **JWT signed with ES256** (ECDSA P-256) by VA360 LABS. The instance verifies it locally with the public keys embedded in the product — there are **no calls to a licensing server** at runtime.

## Activating the license

```bash
# In your .env
DIDACTA_LICENSE_KEY=eyJhbGciOiJFUzI1NiIs...
```

```bash
docker compose -f docker-compose.alpha.yml up -d didacta
```

The license is read **once at startup** and the resulting state is written to the startup log (`License: active`, `License: community (no key set)`…).

!!! note "Changing the license requires a restart"
    There is no admin screen today for entering the license: it is configured through an environment variable and applied by restarting the container.

## What it contains

The JWT payload includes: license and customer identifiers, the organization, the plan and edition, issue and expiry dates, grace days, the list of purchased **capabilities** and, optionally, restrictions (allowed domains, environment, version) and support data (tier, SLA).

## States

| State | When | Capabilities |
| --- | --- | --- |
| `community` | No `DIDACTA_LICENSE_KEY` | None — full Community. |
| `active` | A valid, current license | Those in the payload. |
| `grace` | Expired, within the grace period (30 days by default) | Those in the payload, with a notice. |
| `expired` | Expired, grace period exhausted | None. |
| `invalid` | Invalid signature or payload | None (gated endpoints return 401/402). |
| `dev` | `DIDACTA_DEV_BYPASS=true` (outside production only) | **All of them** — the development bypass. |

The instance warns 30 days before expiry. The state can be checked without authentication at `GET /api/license` (status, capabilities, notices, expiry — never secrets).

## Verification

- A fixed **ES256** algorithm; issuer `didacta.io`, audience `didacta-runtime`.
- Public keys shipped with the product (`packages/license-sdk`), selected by the token's `kid`.
- The payload is also validated against a strict schema; any failure leaves the license as `invalid` without breaking the instance.

## Development and testing

`DIDACTA_DEV_BYPASS=true` enables every capability so you can develop and test Enterprise features. The guard is hard: **it is ignored with `NODE_ENV=production`** and it prints a visible warning at startup. The Enterprise license permits using the EE code for development and testing without a subscription; production requires a valid license (see [Licenses](../comunidad/licencias.md)).

## Getting a license

Contact `licensing@didacta.io` or visit [didacta.io](https://didacta.io).
