# The Enterprise license

Didacta's license is a **JWT signed with ES256** (ECDSA P-256) by VA360 LABS. The instance verifies it locally with the public keys embedded in the product — there are **no calls to a licensing server** at runtime.

## Activating the license

There are **two routes**, and the precedence between them is fixed: **the environment always wins over the panel**.

=== "From the panel (recommended)"

    **Administration → License** (`/admin/licencia`, `super_admin` only) shows the current
    status and lets you paste the key, refresh it or delete it.

    - The key is **validated live before being persisted**: an invalid key returns a 400 and
      **does not overwrite the one you already had**.
    - It is stored encrypted as an instance setting and the license is **reloaded hot**:
      **no container restart is needed**.
    - Endpoints: `GET`/`PUT`/`DELETE /api/v1/admin/license` and `POST /api/v1/admin/license/refresh`.

=== "Through an environment variable"

    ```bash
    # In your .env
    DIDACTA_LICENSE_KEY=eyJhbGciOiJFUzI1NiIs...
    ```

    ```bash
    docker compose -f docker-compose.alpha.yml up -d didacta
    ```

    On this route the license is read **at startup**, so **changing it requires restarting**
    the container. The resulting state is written to the startup log (`License: active`,
    `License: community (no key set)`…).

!!! note "The environment wins over the panel"
    If `DIDACTA_LICENSE_KEY` is set, the panel becomes **read-only**: it hides the activation
    form, shows the license with an "set by the operator" badge, and any attempt to edit or
    delete it returns **409**. This is what lets an operator pin the license per deployment
    without anyone changing it from the interface.

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
