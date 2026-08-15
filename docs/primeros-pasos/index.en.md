---
render_macros: true
---

# What Didacta is

Didacta is a **next-generation fair-code LMS** for academies, instructors and organisations that want to run their own training platform with full control.

## The four pillars

**Genuinely modular.**
Every feature is a self-contained module: courses, assessments, certificates, gamification, community, virtual classroom… Install only what you need, with no patches and no themes that break on every upgrade. The module architecture is described in [Modules](../modulos/index.md).

**Fair-code.**
Your platform, your code. Audit it, modify it and deploy it with free internal use under the [Didacta Sustainable Use License v1.0](https://github.com/va360labs/didacta-io/blob/main/LICENSE). No per-user licensing. Commercial distribution, third-party SaaS and white-labelling require an agreement (see [Licenses](../comunidad/licencias.md)).

**Compliance taken seriously.**
Fundae, GDPR and WCAG 2.2 AA are built into the core, not bolted on with third-party plugins. Traceability, audit logging and data export from day one.

**Unobtrusive AI.**
Artificial intelligence that helps without getting in the way: it drafts content, suggests learning paths and summarises activity. You choose the provider: the AI layer is **multi-provider BYOK** — each installation configures its own provider and API key (see [Configuring AI](../configuracion/ia.md)).

## Three ways to fill your academy with students

The three acquisition paths coexist in the same installation and can be combined freely:

1. **You enrol them yourself.** Invite students one by one or in bulk from the admin panel: you pick their [access group](../modulos/catalogo/access-groups.md) as you invite them, they receive an email to set a password and land straight in their courses. From each student's profile you manage their groups, enrollments and removals.

2. **You sell individual courses.** Publish a course, set a price and share your public catalog at `/catalogo`. Visitors pay by card through Stripe without signing up first: their account is created automatically with the email confirmed at checkout and they are enrolled instantly ([mod.billing](../modulos/catalogo/billing.md)). Refunds revoke access on their own.

3. **You sell memberships.** Create plans with any billing period (1–12 months) and currency, with an optional trial. Your public sales page at `/unete` lists the real catalog; on subscribing, the student gets access to every course in the group you define ([mod.subscriptions](../modulos/catalogo/subscriptions.md)). If they stop paying, access is revoked automatically — without touching anything granted by hand.

And if your community requires prior approval, enable [registration with an approval request](../modulos/catalogo/member-registration.md): per-tenant configurable verifiers, evidence for every request and a one-click decision.

## Technology stack

| Layer | Technology |
| --- | --- |
| Backend | Node.js 22 · NestJS 11 · TypeScript |
| Frontend | Next.js 15 (App Router) · React 19 · Tailwind · shadcn/ui |
| Database | PostgreSQL 16 + Row-Level Security · Prisma · pgvector |
| Cache / queues | Redis 7 · BullMQ |
| Object storage | S3-compatible (MinIO in development, any S3 provider in production) |
| Virtual classroom | Zoom (API + Web SDK) |
| Monorepo | Turborepo · pnpm workspaces |

## Project status

Didacta is in **{{ didacta_channel_en|lower }}** ({{ didacta_version }}): between May and July 2026 the product matured while serving its first deployment in real production; since 31 July 2026 the repository is the whitelabel product, and in August 2026 it entered public beta.

- **SemVer** versioning; every release is published as a Docker image tag. **There is no `latest` tag**: always pin a specific version.
- Change history in the [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md).

## Next steps

- [Editions: Community, Enterprise and Cloud](ediciones.md)
- [Platform architecture](arquitectura.md)
- [Install Didacta in 5 minutes](../instalacion/docker-compose.md)
