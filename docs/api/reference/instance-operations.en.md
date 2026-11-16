# Reference — Instance and control plane

What operates **the whole installation**, not one classroom: the list of organizations, the provisioning credential, support access, the subscription gate, the license and the instance settings. It is the surface used by an external **control plane** —Didacta Cloud, or a script of yours if you self-host— and by a `super_admin` from the panel. Every route hangs off `/api/v1`.

**Auth legend**: `super_admin` = a super_admin session · `super_admin or provisioning` = it also accepts the [provisioning credential](../authentication.md#3-provisioning-credential) (`Authorization: Provisioning didprov_…`). Those marked **Cloud** only exist when the operator turns the feature on (payouts platform mode or video upload); without it the route answers 404.

!!! info "The credential's whitelist"
    The provisioning credential **only opens** the routes marked "provisioning" on this page (plus `GET /admin/system/health-detail`). Any other route answers `403 AUTH_PROVISIONING_ROUTE_NOT_ALLOWED`, even if the credential is valid. Renaming an organization, seeing the license capacity or touching the license require a super_admin with a session.

## Organizations — `/admin/tenants`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/admin/tenants` | super_admin or provisioning | Lists all organizations (with user, course and domain counts). |
| GET | `/admin/tenants/capacity` | super_admin | Number of organizations and whether the license allows creating more: without `feat:multi_tenant.real` the limit is **1**. |
| GET | `/admin/tenants/usage?asOf=` | super_admin or provisioning | **Active members per organization** at a cutoff date (`asOf`, ISO-8601, default now; a future date gives 400). It is the contractual definition of usage —the same one on Didacta Cloud's pricing page—. Counts: users with access, active and not deleted (instructors, auditors and company managers **do** count). Does not count: `tenant_admin`/`super_admin`, never-accepted invitations (`PENDING`), suspended, deactivated and deleted. Returns `[{ tenantId, slug, activeMembers, asOf }]`. |
| GET | `/admin/tenants/:id` | super_admin or provisioning | Detail of one organization. |
| POST | `/admin/tenants` | super_admin or provisioning | Creates an organization + first `tenant_admin` + primary domain and sends the welcome email (the link points to **that** classroom's domain). **402** when exceeding the Community limit without `feat:multi_tenant.real`. See the body below. |
| PATCH | `/admin/tenants/:id/status` | super_admin or provisioning | `{ status: ACTIVE \| SUSPENDED \| ARCHIVED }`. `SUSPENDED` and `ARCHIVED` invalidate all of its users' sessions. |
| PATCH | `/admin/tenants/:id` | super_admin | Renames (`{ name }`); affects the emails' signature ("Team {name}"). Does not invalidate sessions. |
| PATCH | `/admin/tenants/:id/signups` | super_admin or provisioning | Freezes or unfreezes new sign-ups: `{ frozen, reason }` (`reason` 5 to 280 characters; the visitor who gets left out is shown it). With sign-ups frozen, single invitation, bulk upload, public registration and new-student checkout give **409 `TENANT_SIGNUPS_FROZEN`**. It is an explicit switch: the core does not count members and never turns it on by itself. |
| PUT | `/admin/tenants/:id/subscription-gate` | super_admin or provisioning | Opens, puts on trial or closes the [subscription gate](#subscription-gate). |
| POST | `/admin/tenants/:id/domains` | super_admin or provisioning | Adds a domain (`{ hostname }`). |
| DELETE | `/admin/tenants/:id/domains/:hostname` | super_admin or provisioning | Removes a domain; the primary one cannot be removed. |

**`POST /admin/tenants` body**: `slug` (≤64), `name` (≤120), `adminEmail`, `adminName?`, `primaryHostname`, and backward-compatible optionals `simpleMode` (without it, it is born **on** if the control plane creates it and **off** if a super_admin does), `locale` (`es`, `en`, `pt` or a tag such as `pt-BR`, stored reduced; a language with no catalog gives 400; it also sets the first administrator's language) and `payoutsCountry` (the bank country declared at sign-up, ISO alpha-2 or `OTHER`; only used to preselect in the payouts card).

```bash
curl -X POST https://your-instance/api/v1/admin/tenants \
  -H "Authorization: Provisioning didprov_..." \
  -H "Content-Type: application/json" \
  -d '{
    "slug": "demo",
    "name": "Demo Academy",
    "adminEmail": "admin@example.com",
    "primaryHostname": "demo.example.com",
    "locale": "en"
  }'
```

## Subscription gate

A dimension **separate** from the organization's state. `ACTIVE` says the classroom exists and is served; the gate says whether, in addition, its content is open, on trial or paused. A classroom whose trial has ended stays `ACTIVE` on purpose: its administrator must be able to sign in, see it is paused and pay. The core **is not a policy**: it does not count days or look at plans; whoever moves the gate is the control plane or a super_admin. A self-hosted installation that never receives this call stays `OPEN`.

`PUT /admin/tenants/:id/subscription-gate` with `{ state, trialEndsAt?, actionUrl? }`:

| `state` | What happens | Fields |
|---|---|---|
| `OPEN` | Everything as usual. Clears `trialEndsAt` and `actionUrl`. | — |
| `TRIAL` | The classroom works and its administrators see a banner with the days left and a "Choose plan" button → `actionUrl`. | `trialEndsAt` (ISO-8601 with offset) and `actionUrl` (**https**) required. |
| `LOCKED` | A pause screen for everyone and **402 `SUBSCRIPTION_LOCKED`** on the API, except sign-in, `/me/profile`, `/me/modules`, branding, probes, incoming webhooks and organization routes. | `actionUrl` (**https**) required. |

A `TRIAL` whose `trialEndsAt` has passed behaves like `LOCKED` even if the call is not repeated. The call is idempotent: repeating it changes neither `updatedAt` nor leaves another audit-log row. `GET /auth/tenant-context` shows the visitor only whether the classroom is paused (`subscriptionGate.state: LOCKED`); the trial banner is seen only by administrators, in `/me/modules`.

## Support access

A temporary window so the support team can enter a customer's classroom without impersonating anyone.

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/admin/tenants/:id/support-access` | super_admin or provisioning | Opens the window. Body `{ reason, ttlSeconds? }`: `reason` is mandatory (10–280 characters) —it is what the customer reads in **their** audit log—; `ttlSeconds` may ask for **less** than 15 minutes, never more (minimum 60 s). Returns a **single-use** token and the link to redeem it. |
| DELETE | `/admin/tenants/:id/support-access/:grantId` | super_admin or provisioning | Cuts the window before it expires and closes its session. Idempotent: the second time it returns `revoked: false`. |

- Redemption (`POST /auth/support-access/redeem`, see [Core](core.md#support-access-authsupport-access)) opens a session as a **support user of the classroom's own**, visible in its member list; never as a real person. The session has no refresh token and expires with the grant.
- Two rows are left in the **classroom's** audit log: the grant (with the reason and who opened it) and the redemption. The customer reads them without asking permission.
- While the session lives, the classroom shows a permanent notice that cannot be dismissed. A support session cannot export personal data (`/admin/sales/export` gives 403 `SALES_SUPPORT_SESSION_FORBIDDEN`).

## Provisioning credential — `/admin/provisioning-credential` (super_admin)

| Method | Route | What it does |
|---|---|---|
| POST | `/admin/provisioning-credential` | Issues the instance's credential. Returns `{ id, token, preview, createdAt }`; the `didprov_…` token is **seen only this once**. Issuing another **revokes the previous one** (at most one is alive). |
| DELETE | `/admin/provisioning-credential` | Revokes the live credential (idempotent): `{ revoked, id }`. |

How it is used and what it covers: [Authentication](../authentication.md#3-provisioning-credential).

## Platform payouts — pool routes (Cloud)

They only exist with the **platform mode** on (`PAYMENTS_PLATFORM_STRIPE_SECRET_KEY` in the API's environment); on any other installation they answer 404. The control plane uses them to know how much is being collected and to set the commission. What each creator sees is in [Payments, live classroom and AI](payments-live-ai.md#platform-payouts-modulespayouts-cloud).

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/admin/tenants/payouts/volume?from=&to=` | super_admin or provisioning | Volume collected by the platform, **per classroom, day (UTC of the charge) and settlement currency**, between `from` and `to` (both included, `YYYY-MM-DD`, max 366 days). A sale counts once its split is known: re-request the last few days. It never adds different currencies together. |
| GET | `/admin/tenants/payouts/countries` | super_admin or provisioning | Countries (ISO alpha-2) where the platform can pay creators: `PAYMENTS_PLATFORM_SUPPORTED_COUNTRIES` or the default list. A single source for Cloud and for the creator sign-up. |
| PUT | `/admin/tenants/:id/payouts-settings` | super_admin or provisioning | Sets a classroom's **operator commission**: `{ feeBps }` in basis points, 0 to 3000 (500 = 5 %). Idempotent; recorded in the audit log. |
| GET | `/admin/tenants/:id/payouts` | super_admin or provisioning | The classroom's payment mode, connected-account status, commission and open debt. |

## Video — pool routes (Cloud)

They only exist with video upload configured (`BUNNY_STREAM_API_KEY`). The creator's side is in [Learning → Video](learning.md#video).

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/admin/tenants/:id/video` | super_admin or provisioning | Hours quota, last activity, inactivity warnings sent, purge and backups (count and bytes) of a classroom's uploaded video. |
| POST | `/admin/tenants/:id/video-backups/purge` | super_admin or provisioning | **Erasure (GDPR)**: deletes right now, permanently, all Google Drive copies of the classroom's videos. Idempotent. Returns `{ deleted, failed }`. |

## License — `/admin/license` (super_admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/license` | Current license state of the installation. |
| PUT | `/admin/license` | Activates a license: `{ key }`. Validates the signature before saving and rejects it if invalid. |
| DELETE | `/admin/license` | Removes the license saved in the panel (back to Community, or to the environment's if there is one). |
| POST | `/admin/license/refresh` | Re-evaluates the state (expiry, grace) without changing the key. |

The public state —no secrets— is at `GET /api/license`. What each capability unlocks: [Enterprise → Capabilities](../../enterprise/capabilities.md).

## Instance settings — `/admin/instance-settings` (super_admin)

Global `scope`/`key` pairs of the installation (not of a classroom). Secrets are encrypted at rest and **never return their value**, only `hasValue`.

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/instance-settings` · `/:scope` | Lists all instance settings (metadata, no secret values) · those of one scope. |
| GET | `/admin/instance-settings/:scope/:key` | Reads a setting. |
| PUT | `/admin/instance-settings/:scope/:key` | Creates or updates (`{ value, isSecret }`). |
| DELETE | `/admin/instance-settings/:scope/:key` | Deletes the setting. |

## Installation status and registration

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/admin/system/health-detail` | super_admin, tenant_admin or provisioning | Consolidated state of DB, Redis, S3, SMTP, the two queues and the deployed version (on-call diagnostics). |
| GET | `/admin/registry/status` | super_admin | State of the installation's opt-in registration with the Didacta team: whether it is registered, opt-in date, last telemetry send and whether there is a connection. |
| POST | `/admin/registry/opt-in` | super_admin | Registers the installation and turns on sending aggregated telemetry (no personal data). `acceptTerms: true` is mandatory. |
| DELETE | `/admin/registry/opt-in` | super_admin | Opt-out (GDPR): marks the registration as cancelled and requests remote deletion (best-effort if the service does not respond). |

Registration is an **instance** decision, not a classroom one. See [Telemetry](../../configuration/telemetry.md).

## Marketplace modules

Installing, uninstalling and inspecting signed ZIP packages (`/admin/modules/install`, `/admin/modules/installed…`) are `super_admin` operations on the instance and are in [Administration → Modules](administration.md#modules-adminmodules). Once installed, a third-party module answers **everything under `/modules/<slug>/…`** through a generic dispatcher (`ALL /modules/*`): authentication is optional in the dispatcher and each of the module's routes decides whether it requires a session. Each module's UI bundle is served without authentication at `GET /modules/:slug/ui/:surface.js`.
