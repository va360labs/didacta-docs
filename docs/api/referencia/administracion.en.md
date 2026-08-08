# Reference — Administration

Back-office endpoints. Every route hangs off `/api/v1`. **Auth**: `admin` = `tenant_admin` or `super_admin` with a Bearer JWT; some blocks require strict `super_admin`; those marked <span class="didacta-chip didacta-chip--ee">EE</span> additionally require an Enterprise capability (no license → **402**).

## Organizations — `/admin/tenants` (super_admin only)

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/tenants` · `/:id` | Lists every tenant · detail. |
| GET | `/admin/tenants/capacity` | The number of tenants and whether the license allows creating more. |
| POST | `/admin/tenants` | Creates a tenant + the first `tenant_admin` + the primary domain, and sends a welcome email. **402** when the Community limit is exceeded without `feat:multi_tenant.real`. |
| PATCH | `/admin/tenants/:id` · `/:id/status` | Renames it · changes its status (`SUSPENDED`/`ARCHIVED` invalidate every session of its users). |
| POST · DELETE | `/admin/tenants/:id/domains[/:hostname]` | Adds a domain · removes it (the primary one cannot be removed). |

## Users — `/admin/users` (admin, scoped to the token's tenant)

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/users` | Paginated list with filters (`search`, `status`, `role`, `externalSource`, `page`, `limit`). |
| GET | `/admin/users/:id` | Detail with roles and recent sessions. |
| POST | `/admin/users` | Invites: creates a `PENDING` user + an email to set the password. `{ email, name?, role, accessGroupId? }`. |
| PATCH | `/admin/users/:id/status` | `ACTIVE`/`SUSPENDED`/`DEACTIVATED` (suspending invalidates sessions; you cannot suspend yourself). |
| POST · PATCH | `/admin/users/:id/roles` · `/roles/remove` | Assigns / removes a role. Assignable: `tenant_admin`, `formador`, `alumno`, `auditor`, `empresa_manager` (never `super_admin`). |
| POST | `/admin/users/:id/resend-invite` | Resends the invitation. |

## Invitations — `/admin/invitations` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/invitations/summary` · `/admin/invitations` | Counters and a status listing (filters `invitados`/`activados`/`sin-enviar`/`sin-acceso`). |
| POST | `/admin/invitations/send-batch` | Batched background sending (`size`, `emails?` to prioritise, `pauseMs?`). Idempotent: nobody receives it twice. |

## Modules — `/admin/modules`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/admin/modules` | admin | Available modules with their state and dependencies (`?tenantId=` for another tenant, super_admin only). |
| POST | `/admin/modules/:name/enable` · `disable` | admin | Enables (idempotent) · disables (`?force=true` cascades). Errors: 409 active dependents, 422 core module. |
| POST | `/admin/modules/install` | super_admin | Installs a ZIP package (body = the raw ZIP, `Content-Type: application/zip`). Typed errors: 413 too large, 422 signature/lint/boot, 403 untrusted vendor, 412 incompatible core, 409 reserved name or already installed. |
| GET | `/admin/modules/installed[/:name[/routes]]` | super_admin | Installed third-party modules · detail · exposed routes. |
| DELETE | `/admin/modules/installed/:name` | super_admin | Uninstalls it (deregisters routes, deletes the record and the blob) → 204. |

## Keys, images and metrics (admin)

| Method | Route | What it does |
|---|---|---|
| GET · POST · DELETE | `/admin/api-keys[/:id]` | API keys for the **whole tenant** (creating returns the token exactly once; DELETE revokes → 204). |
| GET · POST | `/admin/images/inventory` · `optimize` | Image inventory with estimated savings · re-optimises up to 50 per batch. |
| GET | `/admin/stats?range=all\|7d\|30d` | Active users, courses, enrollments, certificates, completion rate. |
| GET | `/admin/metrics/business` | Business KPIs: 30-day NPS, sales, overdue payments, sign-ups/cancellations, AI tutor usage, activity. |
| GET | `/admin/system/health-detail` | Consolidated status of the database, Redis, storage, SMTP and outbox (on-call diagnostics). |
| GET | `/admin/rate-limit/info` | The active rate limit tier and effective limits (informational, never a 402). |

## SMTP — `/admin/tenant-settings/smtp` (admin)

| Method | Route | What it does |
|---|---|---|
| GET · PUT | `/admin/tenant-settings/smtp` | Configuration without the password (`hasPassword`, `verifiedAt` flags) · create/edit (an empty password keeps the stored one; credentials encrypted with AES-256-GCM). |
| POST | `/admin/tenant-settings/smtp/test` | Sends a test email with the tenant's SMTP; on success it stamps `verifiedAt`. The MTA's real error is surfaced to the admin. |
| POST | `/admin/tenant-settings/smtp/test-template` | Sends the real email of a catalog template, with variables. |

## Corporate identity <span class="didacta-chip didacta-chip--ee">EE</span>

| Method | Route | Capability | What it does |
|---|---|---|---|
| GET · PUT · DELETE | `/admin/sso/oidc/config` | `feat:sso.oidc` | OIDC configuration (the `clientSecret` is encrypted and never comes back in a GET; sending it in a PUT rotates it). |
| POST | `/admin/sso/oidc/test-discovery` | `feat:sso.oidc` | OIDC Discovery against the issuer. |
| GET · PUT · DELETE | `/admin/sso/saml/config` | `feat:sso.saml` | SAML configuration + the SP URLs (`entityId`, `acsUrl`, `metadataUrl`) to paste into the IdP. |
| POST | `/admin/sso/saml/test-connection` | `feat:sso.saml` | Validates the PEM certificate and the URL (SAML has no discovery). |
| GET · POST · DELETE | `/admin/scim/token` | `feat:scim` | Status · generates a new token (`scim_…`, **shown exactly once**; it replaces the previous one) · revokes it. |
| GET · PUT | `/admin/mfa-policy` | PUT: `feat:mfa.enforcement` | The tenant's MFA policy (`requiredForAll`, `gracePeriodDays` 1-90). The GET is unrestricted so the upsell can be rendered. |
| GET | `/super/users` | `feat:multi_tenant.real` (super_admin only) | Users across **every** tenant, with filters. |
| GET · POST · PATCH · DELETE | `/admin/custom-domains[...]` | `feat:custom_domains` | Custom domains: creation (generates `cnameTarget` + a token), verification, suspension/reactivation, deletion. |
| GET · POST · DELETE | `/admin/webhooks/dead-letter[...]` | `feat:api.webhooks.high_throughput` | The webhook dead-letter queue: list · retry (202) · discard (204). |

WP-SSO is Community: `GET/PUT/DELETE /admin/sso/wp/config` (admin, no capability required).

## AI — `/admin/ai` (admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/ai/providers/catalog` · `/admin/ai/providers` | Available providers (openai, anthropic, gemini, openrouter, mistral, groq, ollama, voyage) · the tenant's configurations (without keys). |
| PUT · DELETE | `/admin/ai/providers/:purpose` | Configures `chat` or `embed` (`{ provider, model?, apiKey, baseUrl?, … }`; the key is encrypted) · deletes it (falling back to the global default). |
| GET · POST | `/admin/ai-tutor/answers[/:messageId/review]` | Review of the tutor's answers: a filtered listing · marking one correct or corrected (a correction becomes validated knowledge). |
| GET · POST · PATCH · DELETE | `/admin/ai-tutor/corrections[/:id]` | Validated knowledge: CRUD (changing the question recomputes the embedding). |
| GET | `/admin/ai-tutor/report/monthly?mes=YYYY-MM` | Monthly report of questions by topic and volume. |
| POST | `/admin/ai-tutor/courses/:courseId/index` · `/admin/ai-tutor/reindex-all` | Reindexes one course · every published course (the RAG backfill). |

## Templates, moderation and registration

| Method | Route | What it does |
|---|---|---|
| GET | `/admin/notifications/templates/keys` · `catalog` · `/admin/notifications/templates?key=` | Known keys · the catalog with variables · the tenant's overrides. |
| PUT · DELETE | `/admin/notifications/templates/:key` | Override per `(channel, locale)` · deletion (with no query parameters it deletes every override for that key). |
| GET · POST | `/admin/users/:userId/restrictions` | Sanction history · sanction a user (`{ scopes[], reason, expiresAt? }` — the user can still sign in and read, but not contribute). |
| POST | `/admin/users/:userId/restrictions/:id/lift` | Lifts the sanction (it stamps `liftedAt`, it does not delete). |
| GET | `/admin/users/:userId/dossier` | The full dossier (identity, purchases, training, activity, sanctions). **Every lookup is audited.** |
| GET | `/admin/restrictions/scopes` · `active?userIds=csv` | Sanctionable areas · active sanctions in bulk (max. 200). |
| GET · POST · DELETE | `/admin/registry/status` · `opt-in` | **super_admin.** Opt-in registration of the installation with the Didacta team (`acceptTerms: true` is mandatory) · GDPR opt-out with remote erasure. An instance-level decision, not a tenant one. |

## Fundae (admin; the instructor role has no access)

| Method | Route | What it does |
|---|---|---|
| GET · POST · PATCH · DELETE | `/admin/fundae/companies[/:id]` | Subsidised companies: Spanish tax ID with checksum (immutable once created), CCC, credit. 409 on deletion if it has active groups. |
| GET · POST · DELETE | `/admin/fundae/companies/:companyId/rlpt-notices[/:id]` | RLPT notices (PDF/image ≤10 MiB into the Evidence Vault with its hash). The statutory 15-day period is calculated automatically. |
| GET · POST · PATCH | `/admin/fundae/groups[/:id]` | Subsidised groups (states DRAFT→ACTIVE→CLOSED/CANCELLED). |
| POST | `/admin/fundae/groups/:id/start` · `close` · `cancel` · `finalize` | Transitions: `start` validates RLPT and credit; `close` debits the credit; `finalize` computes APTO/NO_APTO (75% threshold, with `preview`). |
| GET | `/admin/fundae/groups/:id/start-xml` · `end-xml` · `audit-zip` | Start XML · end XML (named participants + costs) · the complete audit ZIP with a SHA-256 manifest. |
| GET · POST · PATCH · DELETE | `/admin/fundae/groups/:id/costs[/:costId]` | Costs allocated to the group (locked once the group is closed). |
| GET · POST · PATCH · DELETE | `/admin/fundae/groups/:groupId/participants[/:id]` | Named participants (+ `bulk-enroll` from the action's course). Soft delete → `REMOVED`, for Fundae traceability. |
