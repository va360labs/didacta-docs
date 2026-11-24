# Endpoint reference

An **endpoint-by-endpoint** reference for the Didacta API, extracted from the real code of version 0.1.0-beta.24 (about 130 controllers and more than 700 endpoints), plus the membership-level, per-level content and academy-address routes of 0.1.0-beta.27, organised into six areas:

<div class="didacta-cards">
  <a href="core/"><h3>🔐 Core and cross-cutting</h3><p>Auth, MFA, API keys, my account, setup, public license, the classroom's public site, storage, auditing, SSO, SCIM, external enrollment, MCP agents and webhooks.</p></a>
  <a href="administration/"><h3>🛠️ Administration</h3><p>Users, invitations, classroom, simple mode, modules, sales, sales emails, SMTP, Stripe, corporate identity (EE), AI, moderation and Fundae.</p></a>
  <a href="instance-operations/"><h3>🧭 Instance and control plane</h3><p>Organizations, the provisioning credential, support access, the subscription gate, the license, instance settings and Cloud routes.</p></a>
  <a href="learning/"><h3>🎓 Learning</h3><p>Courses, enrollments and progress, drip, SCORM, learning paths, assessments, certificates, access groups, video and Fundae.</p></a>
  <a href="community/"><h3>👥 Community and people</h3><p>Community, SSE messaging, gamification and retention, resources, surveys, referrals, theming, registration, membership and help and suggestions.</p></a>
  <a href="payments-live-ai/"><h3>💳 Payments, virtual classroom and AI</h3><p>Billing, subscriptions, payment connections, platform payouts (Cloud), live Zoom and the three AI modules.</p></a>
</div>

## How to read the reference

- Every route hangs off **`/api/v1`** except those marked with ⚠ (`/healthz`, `/metrics`, `/api/license`, `/scim/v2`).
- The **Auth** column states the minimum required: `Public` (no session, tenant resolved by domain), `Bearer` (any authenticated user), `instructor+`/`staff`, `admin` (tenant_admin or super_admin), `super_admin`, `super_admin or provisioning` (it also accepts the [provisioning credential](../authentication.md#3-provisioning-credential)), `ApiKey` + scope, an agent key, or a webhook signature.
- Edition marks: <span class="didacta-chip didacta-chip--ee">EE</span> = requires an Enterprise capability (without a license, **402** with the capability in the body). **Cloud** = only exists when the operator turns the feature on (payouts platform mode, video upload); without it the route answers 404.
- Endpoints of a module disabled for the tenant return **403** `TENANT_MODULES_MODULE_NOT_ACTIVE`; with the classroom paused (subscription gate), **402** `SUBSCRIPTION_LOCKED`. More in [Conventions](../conventions.md).
- Domain errors carry a stable `code` field (`COURSE_SLUG_EXISTS`, `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING`…) designed to be programmed against.

!!! tip "The fine-grained contract lives in your Swagger"
    This reference covers routes, auth, purpose, main bodies and errors. The exact request/response schemas of **your deployed version** are always in your instance's Swagger: `https://your-instance/api/docs` (JSON at `/api/docs.json`).

!!! note "What it does not cover"
    Third-party modules installed from the marketplace publish their own routes under `/modules/<slug>/…` and are documented in their package. The installation's maintenance tasks (rotating the encryption key, repairing access after refunds) are command-line tools, not endpoints.
