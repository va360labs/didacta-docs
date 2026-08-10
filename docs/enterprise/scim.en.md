# SCIM 2.0 provisioning

Didacta exposes a **SCIM 2.0** server (RFC 7643 and RFC 7644) so your identity provider —Okta, Microsoft Entra ID (Azure AD), Auth0, Google Workspace— can create, update and deactivate users automatically, with nobody keeping two lists in sync by hand.

It is an Enterprise capability: `feat:scim`. Without it the user endpoints return **402** and the IdP cannot provision.

!!! warning "Read what is not supported first"
    The scope is **deliberately limited to users**. Groups, `bulk`, `changePassword`, `sort` and `etag` do not exist here, and a connector configured to use them will fail. The full list is in [What is not supported](#what-is-not-supported); it is worth reading before you configure the connector rather than during the first sync.

## What it supports, at a glance

This is literally what `GET /scim/v2/ServiceProviderConfig` declares, which is what the IdP reads before letting you save the configuration:

| Feature | Status |
| --- | --- |
| `User` resource | **Yes** — list, read, create, update and delete |
| `Group` resource | **No** |
| `patch` (PatchOp) | **Yes**, over a limited set of paths |
| `filter` | **Yes**, only `userName eq "…"` (`maxResults`: 200) |
| `bulk` | **No** (`maxOperations: 0`) |
| `changePassword` | **No** |
| `sort` | **No** |
| `etag` | **No** |
| Authentication | `oauthbearertoken` — a static token **per organization** |

## The endpoint URL

```text
https://<your-domain>/scim/v2
```

- **It has no `/api/v1` prefix.** SCIM is one of the few product routes outside it (along with the probes, `/metrics` and `/api/license`), because many connectors reject a base URL that does not end in exactly `/scim/v2`.
- **It is the same for every organization**: the token is what identifies the organization, not the URL. If you use custom domains, any domain that resolves to the instance works.
- The panel shows it already built with your domain, ready to copy.

## Issuing the token

**Administration → Security → Provisioning (SCIM)** (`/admin/scim`). Only `super_admin` and `tenant_admin` can manage it.

1. Click **Generate token**. It creates a secret with the `scim_` prefix and 43 random characters (256 bits of entropy).
2. The token is shown **exactly once**. Copy it and paste it into the IdP right then.
3. From that point the panel only shows the **prefix** (the first 13 characters), the creation date and the status.

Only a **SHA-256 hash** is stored in the database: the plaintext token is never persisted anywhere, so nobody —not even someone with database access— can recover it. If you lose it, the only way out is generating a new one.

| Action | Effect |
| --- | --- |
| **Generate** | Creates the token and reveals it once. |
| **Rotate** | Generates a new one and **revokes the previous one immediately**. |
| **Revoke** | Deletes the token: the IdP starts receiving **401** right away. |

The same three buttons exist as an API, with an admin JWT plus `feat:scim`: `GET` · `POST` · `DELETE /api/v1/admin/scim/token`.

!!! warning "One active token per organization"
    Each organization has **a single SCIM token**. Generating a new one invalidates the previous one with no overlap period: if you had two connectors pointing at the same organization, the second one would stop working the moment you rotate.

!!! note "«Not used by the IdP yet»"
    The panel shows a last-used field that is **never populated today**: the server does not record the date of the last authenticated SCIM request, so the label stays on "not used yet" forever. To check that the IdP is getting through, look at the audit log (`scim.user.*`), not at that field.

## Authentication

Every request, **including the discovery ones**, carries:

```http
Authorization: Bearer scim_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

!!! note "Discovery asks for the token too"
    RFC 7644 §4 allows `ServiceProviderConfig`, `ResourceTypes` and `Schemas` to be anonymous. In Didacta they **are not**: the guard covers every endpoint, so no configuration leaks before a credential exists. Set the token in the IdP **before** pressing "Test connection", or the test will return 401. Okta and Entra ID accept this without trouble.

A missing `Authorization` header, a scheme other than `Bearer` (`Basic`, `ApiKey`…), an empty token and an unrecognised token (revoked, rotated or from another instance) all return **401**.

The token resolves the organization and **every query is scoped to it**: one organization's token cannot see, modify or find another organization's users. A `GET` for an id that exists in a different organization returns 404, not 403.

## Endpoints

| Method | Route | What it does | Capability |
| --- | --- | --- | --- |
| `GET` | `/scim/v2/ServiceProviderConfig` | Declares what the server supports. | — |
| `GET` | `/scim/v2/ResourceTypes` | A single type: `User`. | — |
| `GET` | `/scim/v2/Schemas` | Attributes of the core `User` schema. | — |
| `GET` | `/scim/v2/Users` | Paginated list, with an optional filter. | `feat:scim` |
| `POST` | `/scim/v2/Users` | Creates a user (**201**). | `feat:scim` |
| `GET` | `/scim/v2/Users/{id}` | Reads a user. | `feat:scim` |
| `PATCH` | `/scim/v2/Users/{id}` | Applies a `PatchOp`. | `feat:scim` |
| `DELETE` | `/scim/v2/Users/{id}` | Deletes the user (**204**). | `feat:scim` |

The three discovery endpoints are **not** gated by the license (they ask for a token, but not for a capability): that way the IdP administrator can validate the URL even before the license is active. The five `Users` operations are gated.

Any other route under `/scim/v2` —`/Groups`, `/Me`, `/Bulk`— **does not exist**: the IdP gets a **404** with the generic API error, not a SCIM body.

## How attributes are mapped

Didacta adds no tables for SCIM: the users the IdP creates are the very same ones you see in **Administration → Users**.

| SCIM attribute | Field in Didacta | Notes |
| --- | --- | --- |
| `userName` | Email | **Required, and it must be a valid email address.** It is normalised to lowercase. |
| `emails` | Email | Only used as a fallback when `userName` is not an email: the one marked `primary` first, then the first in the list. Responses always return a single email with `type: work`, `primary: true`. |
| `name.givenName` + `name.familyName` | Name | Didacta stores **a single text field**. On read it is split with a heuristic: the last word is `familyName` and the rest is `givenName` (`Juan Carlos Pérez` → `Juan Carlos` + `Pérez`). |
| `name.formatted` | Name | If present, it wins over `givenName`/`familyName` on creation. |
| `displayName` | Name | Fallback when `name` is absent. In responses it is the name, or the email if the user has no name. |
| `active` | Status | See the next table. |
| `locale` | Language | Free text (`es-ES`, `en-GB`…). If absent on creation it defaults to `es-ES`. |
| `externalId` | — | **Not persisted.** Responses return a copy of the internal `id` instead, so the IdP can store a single identifier. |
| `meta.location` | — | A **relative** path (`/scim/v2/Users/{id}`), not absolute: the instance does not pin a domain. |
| `password` | — | Ignored. Didacta does not accept passwords over SCIM. |

## `active` and the user lifecycle

| Operation | Resulting status |
| --- | --- |
| `POST` with `active: true` (or with the field absent) | **Pending** — the user exists but has not signed in yet. |
| `POST` with `active: false` | **Deactivated**. |
| `PATCH active: false` | **Deactivated**, from any status. |
| `PATCH active: true` on a pending or deactivated user | **Active**. |
| `PATCH active: true` on a **suspended** user | **Stays suspended**. |

On read, `active` is `true` for active and pending users, and `false` for suspended and deactivated ones.

Three consequences worth having clear before the first sync:

- **A manual suspension wins.** If an administrator suspends somebody from the panel, a `PATCH active: true` from the IdP will **not** reinstate them: they have to be lifted from **Administration → Users**. The other direction always works: the IdP can cut anyone's access.
- **Deactivating closes the session immediately.** A `PATCH active: false` deletes the user's live sessions, so a cut-off from the IdP takes effect at once instead of waiting for a token to expire.
- **Creating a user sends no email and grants no role.** A SCIM creation does not send an invitation (the user is expected to arrive over SSO) and assigns no roles or groups: that is still done from **Administration → Users**. A freshly provisioned user exists, but cannot do anything yet.

`DELETE` performs a **soft delete**: it marks the user as deleted, leaves them deactivated and removes all their sessions. They stop appearing in `GET /Users` and their `id` starts returning 404. IdPs rarely use it —they prefer `PATCH active: false`— but it is there for GDPR erasure requests.

## Listing: pagination and filter

```http
GET /scim/v2/Users?startIndex=1&count=50&filter=userName%20eq%20%22ana@acme.com%22
```

| Parameter | Values | Notes |
| --- | --- | --- |
| `startIndex` | Integer **from 1**, defaults to `1` | It is 1-based, as RFC 7644 §3.4.2 requires. A `0` returns 400. |
| `count` | `0`–`200`, defaults to `50` | Above 200 it returns 400. `0` is accepted (IdPs use it to ask for the total only). |
| `filter` | Only `userName eq "value"` | The value is compared against the email **case-insensitively**. |

Any other filter (`displayName co "…"`, `active eq true`, expressions with `and`…) returns **400** with `scimType: invalidFilter`. It is the only filter most IdPs need: they use it to check whether the user already exists before creating them.

The order is always **by creation date, ascending**, and cannot be changed: `sort` is not supported. The `attributes` and `excludedAttributes` parameters are ignored; the full resource is always returned.

## `PATCH`: supported operations

The body is a standard `PatchOp` with a maximum of **20 operations** per request:

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
  "Operations": [
    { "op": "replace", "path": "active", "value": false }
  ]
}
```

| `path` | Accepted `op` | Value |
| --- | --- | --- |
| `active` | `replace`, `add`, `remove` | Boolean. `remove` means `false`. |
| `userName` | `replace`, `add` | A valid email. If another user in the organization already uses it → **409**. `remove` → 400. |
| `displayName` | `replace`, `add`, `remove` | Text. `remove` clears the name. |
| `name.givenName` | `replace`, `add`, `remove` | Text. Rebuilds the name keeping the current family name. |
| `name.familyName` | `replace`, `add`, `remove` | Text. Rebuilds the name keeping the current given name. |
| `locale` | `replace`, `add`, `remove` | Text. `remove` returns it to `es-ES`. |
| *(no `path`)* | `replace`, `add` | An object with `active`, `userName`, `locale`, `name` or `displayName`: applied as a partial merge. `remove` without `path` → 400. |

Any other path returns **400** with `scimType: invalidPath`. A request whose operations change nothing returns **200** with the user as-is, without writing to the database or leaving an audit entry.

## Error format

SCIM's own errors follow RFC 7644 §3.12: `status` as a string and, on the 4xx that define one, a `scimType`.

```json
{
  "schemas": ["urn:ietf:params:scim:api:messages:2.0:Error"],
  "status": "409",
  "scimType": "uniqueness",
  "detail": "User with userName \"ana@acme.com\" already exists.",
  "statusCode": 409,
  "message": "Conflict Exception"
}
```

!!! note "The two extra fields"
    `statusCode` and `message` are **not SCIM**: the API's error normaliser adds them to every response. Every field the specification requires is present, and SCIM clients ignore attributes they do not know, so in practice they do no harm. Worth knowing if you are diffing the response against the RFC. Responses are also served as `application/json`, not as `application/scim+json`.

| Code | `scimType` | When |
| --- | --- | --- |
| **400** | `invalidFilter` | A `filter` other than `userName eq "…"`. |
| **400** | `invalidPath` | An unsupported `PatchOp` path. |
| **400** | `invalidSyntax` | `replace` without a `path` whose value is not an object. |
| **400** | `invalidValue` | A `userName` that is not an email; an `active` that is not a boolean; a `locale` or `displayName` that is not text. |
| **400** | `mutability` | An attempt to remove `userName`. |
| **400** | `noTarget` | `remove` without a `path`. |
| **401** | — | Missing token, not `Bearer`, empty, or unrecognised. |
| **404** | — | The user does not exist in this organization (or has been deleted). |
| **409** | `uniqueness` | There is already a user with that `userName`. This is the answer to a repeated creation: the IdP should treat it as "already exists", not as a failure. |

Three kinds of response do **not** use the SCIM format, because the API produces them before reaching the controller. If your IdP shows them as raw text, that is expected:

- **402** (missing `feat:scim`): `{ "statusCode": 402, "error": "CapabilityRequiredError", "capability": "feat:scim", … }`.
- **429** (rate limit): includes `retryAfterSeconds` and the `Retry-After` header.
- **Validation 400s** on the body or the query string (for example `count=500`): `{ "statusCode": 400, "message": …, "issues": [ … ] }`.

## What is not supported {#what-is-not-supported}

| Not supported | What happens if the IdP tries |
| --- | --- |
| **Groups** (`/scim/v2/Groups`) | **404.** There is no group or role sync: memberships are still managed in **Administration → Users**. Turn "push groups" off in the connector. |
| **`bulk`** (`/scim/v2/Bulk`) | **404.** The IdP must use individual operations; standard connectors do so by default. |
| **`PUT`** on a user | **404.** Full resource replacement does not exist; use `PATCH`. |
| **`changePassword`** | Passwords are not managed over SCIM. The `password` field of a creation is silently ignored. |
| **`sort`** (`sortBy`, `sortOrder`) | Ignored: the order is always by creation date, ascending. |
| **`etag`** | No `ETag` is emitted and no `If-Match` is honoured: there is no optimistic concurrency control. |
| **Complex filters** | **400** `invalidFilter`. Only `userName eq "…"`. |
| **`/Me`** | **404.** |
| **The `enterprise:2.0:User` extension** | Accepted in the body and ignored: `employeeNumber`, `department`, `manager`… are not stored. |
| **A persisted `externalId`** | Accepted and ignored; the response returns the internal `id` in its place. If your IdP correlates by `externalId` it will work, but against Didacta's identifier. |
| **`attributes` / `excludedAttributes`** | Ignored: the full resource is always returned. |
| **Reverse provisioning** (Didacta → IdP) | Does not exist. The flow always runs from the IdP into Didacta. |

## Rate limit

SCIM requests **count against the instance's public quota**, because a SCIM token is not a user session:

| Plan | Limit |
| --- | --- |
| Community | 30 requests/minute |
| With `feat:api.rate_limit.elevated` | 300 requests/minute |

They can be adjusted with `RATE_LIMIT_COMMUNITY_PUBLIC_PER_MIN` and `RATE_LIMIT_ENTERPRISE_PUBLIC_PER_MIN` (see [Environment variables](../configuracion/variables-de-entorno.md)). With no Redis configured the limiter opens up and counts nothing.

!!! warning "The first sync is the painful one"
    `feat:scim` and `feat:api.rate_limit.elevated` are **different** capabilities. With only the first one, an initial sync of several hundred users can hit the 30 req/min ceiling and get **429**. Connectors retry, but the full import will drag; if you are migrating a large directory, buy the elevated limit too or raise it through the environment variable for the duration of the load.

## Configuring your IdP

Field names vary a little between providers, but the steps are the same:

1. **Create the SCIM application** in the IdP catalogue: look for "SCIM 2.0" or create a custom application with SCIM provisioning enabled.
2. **Paste the base URL** into `SCIM Connector base URL` (Okta), `Tenant URL` (Entra ID) or the equivalent field: `https://<your-domain>/scim/v2`.
3. **Paste the token** into `OAuth Bearer Token` or `Secret Token`, and set the authentication method to **Bearer**.
4. **Map the attributes**: `userName` to the email, `name.givenName` and `name.familyName` to the first and last name, `active` to the user status. Leave everything that is not in the table above unmapped.
5. **Test the connection.** The IdP's "Test connection" button issues a `GET /scim/v2/ServiceProviderConfig`; if it returns 200, you are done.

And before switching it fully on: **disable group synchronisation** in the connector, or the IdP will try to write to `/Groups` and pile up errors.

## Auditing

Every operation lands in the audit log, prefixed so you can filter them at a glance:

| Action | When |
| --- | --- |
| `scim.user.created` | Creation via `POST /Users`. |
| `scim.user.updated` | A `PATCH` that changes something (the ones that change nothing are not recorded). |
| `scim.user.deleted` | `DELETE /Users/{id}`. |
| `scim.token.created` · `scim.token.rotated` · `scim.token.revoked` | Token management from the panel or the admin API. |

Operations arriving over SCIM have no human actor: they are recorded with no author, flagged as coming from the IdP.

## Troubleshooting

| Symptom | Usual cause |
| --- | --- |
| "Test connection" returns **401** | The token is not in the IdP yet (discovery requires it too), it was copied wrong, or you rotated it and the connector still holds the old one. |
| Everything returns **402** | The license does not include `feat:scim`, has expired, or is invalid. Check **Administration → License**. |
| Creation returns **409** | A user with that email already exists in the organization. It is the correct answer to a repeated creation. |
| Creation returns **400** `invalidValue` | `userName` is not an email. Change the IdP mapping so it sends the email address. |
| **429** during the initial load | Public quota exhausted. See [Rate limit](#rate-limit). |
| The IdP piles up group errors | Group synchronisation is still enabled in the connector. It is not supported: turn it off. |
| A user comes back as active after being suspended | The opposite, in fact: suspended users are **not** reinstated over SCIM. If they are active again, somebody reactivated them from the panel. |
| The user exists but cannot sign in | They still need a role and groups: SCIM assigns neither. Do it in **Administration → Users**. |
| The panel says "not used by the IdP yet" even though it works | That field is never updated. Look at the audit log. |

## Related

- [Enterprise capabilities](capabilities.md) — what each capability unlocks.
- [The Enterprise license](licencia.md) — how `feat:scim` gets activated.
- [API authentication](../api/autenticacion.md) — the product's other credential schemes.
- [Reference: core and cross-cutting](../api/referencia/nucleo.md) — the summarised SCIM route table alongside the rest of the API.
