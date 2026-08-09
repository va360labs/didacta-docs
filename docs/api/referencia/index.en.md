# Endpoint reference

An **endpoint-by-endpoint** reference for the Didacta API, extracted from the real code (~100 controllers, ~540 endpoints), organised into five areas:

<div class="didacta-cards">
  <a href="nucleo/"><h3>🔐 Core and cross-cutting</h3><p>Auth, MFA, API keys, my account, setup, license, branding, storage, auditing, SSO, SCIM, external enrollment and webhooks.</p></a>
  <a href="administracion/"><h3>🛠️ Administration</h3><p>Organizations, users, invitations, modules, SMTP, corporate identity (EE), AI, moderation and Fundae.</p></a>
  <a href="aprendizaje/"><h3>🎓 Learning</h3><p>Courses, enrollments and progress, drip, SCORM, learning paths, assessments, certificates, access groups and Fundae.</p></a>
  <a href="comunidad/"><h3>👥 Community and people</h3><p>Community, SSE messaging, gamification, resources, surveys, referrals, theming, registration and membership.</p></a>
  <a href="pagos-directo-ia/"><h3>💳 Payments, virtual classroom and AI</h3><p>Billing, subscriptions, payment connections, live Zoom and the three AI modules.</p></a>
</div>

## How to read the reference

- Every route hangs off **`/api/v1`** except those marked with ⚠ (`/healthz`, `/metrics`, `/api/license`, `/scim/v2`).
- The **Auth** column states the minimum required: `Public` (no session, tenant resolved by domain), `Bearer` (any authenticated user), `instructor+`/`staff`, `admin` (tenant_admin or super_admin), `super_admin`, `ApiKey` + scope, or a webhook signature.
- Enterprise endpoints return **402** with the required capability in the body; endpoints of a module disabled for the tenant return **403**.
- Domain errors carry a stable `code` field (`COURSE_SLUG_EXISTS`, `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING`…) designed to be programmed against.

!!! tip "The fine-grained contract lives in your Swagger"
    This reference covers routes, auth, purpose, main bodies and errors. The exact request/response schemas of **your deployed version** are always in your instance's Swagger: `https://your-instance/api/docs` (JSON at `/api/docs.json`).
