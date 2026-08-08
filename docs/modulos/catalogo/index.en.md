# Module catalog

The 24 modules bundled with Didacta Community, **each with its own detail page**: what it does, how it works, dependencies, data model, API, events and configuration. Those in the **core** category are always active; the rest are enabled per organization.

## Learning (core)

| Module | What it does |
| --- | --- |
| [Courses and catalog](courses.md) (`mod.courses`) | Courses, modules and lessons: the catalog and the instructor's editor, with a publication validation hook. |
| [Player, enrollment and progress](learning.md) (`mod.learning`) | Enrollment, progress, drip content, invitations, SCORM, learning paths and skills. |
| [Quizzes and exams](assessments.md) (`mod.assessments`) | Six question types, auto-grading + manual grading, attempts and thresholds. |
| [Certificates](certificates.md) (`mod.certificates`) | Templates, automatic PDF issuing with an immutable snapshot, and public verification. |
| [Access groups](access-groups.md) (`mod.access-groups`) | Composable entitlements: groups that grant courses, with provenance and refcounting. |

## People and access

| Module | What it does |
| --- | --- |
| [Member registration](member-registration.md) (`mod.member-registration`) *(core)* | Sign-up with composable verifiers (Telegram, OTP) and manual approval with payment lookup. |
| [WordPress SSO](wp-sso.md) (`mod.wp-sso`) | Entry from a WordPress session with a single-use HMAC token. |

## Revenue (core)

| Module | What it does |
| --- | --- |
| [Stripe Checkout](billing.md) (`mod.billing`) | One-off course payments, including a public purchase with no prior account. |
| [Subscriptions](subscriptions.md) (`mod.subscriptions`) | Recurring billing with Stripe: grace periods, dunning, cancellation and the `/unete` membership. |
| [Payment connections](payment-connections.md) (`mod.payment-connections`) | Read-only Stripe/Woo accounts, tiers, a subscription dashboard and an order mirror. |
| [Referral program](referrals.md) (`mod.referrals`) | Commissions on real charges, with a guarantee window and traceable manual payouts. |

## Community and engagement

| Module | What it does |
| --- | --- |
| [Community](community.md) (`mod.community`) *(core)* | Posts, comments, reactions, mentions, spaces, moderation, digest and broadcasts. |
| [Messaging](messaging.md) (`mod.messaging`) | Rooms per space, direct messages and an instructors channel, in real time over SSE. |
| [Points and challenges](gamification.md) (`mod.gamification`) | An auditable points ledger: rules with a daily cap, levels, perks and challenges with proof. |
| [Resource library](resources.md) (`mod.resources`) | Resources grouped in collections, shared by the community, with a download counter. |
| [Surveys and NPS](surveys.md) (`mod.surveys`) | Anonymous surveys triggered at the end of every live class. |

## Artificial intelligence

All three use the [BYOK AI Gateway](../../configuracion/ia.md) — you choose the provider and the key.

| Module | What it does |
| --- | --- |
| [Conversational tutor](ai-tutor.md) (`mod.ai-tutor`) | A per-course tutor with RAG, lesson citations and a quality loop backed by validated knowledge. |
| [Rubric-based grading](ai-grader.md) (`mod.ai-grader`) | Per-criterion score suggestions for open-ended answers; the instructor confirms. |
| [Content generator](ai-content.md) (`mod.ai-content`) | Summaries, flashcards and quizzes from a lesson, always as drafts. |

## Live, compliance and integrations

| Module | What it does |
| --- | --- |
| [Zoom virtual classroom](zoom-live.md) (`mod.zoom-live`) | Live classes with registration, calendar, reminders and reconciled attendance. |
| [Fundae](fundae.md) (`mod.fundae`) | Spanish subsidised training: training actions, RLPT, groups, costs, XML and an audit ZIP. |
| [Visual customization](theming.md) (`mod.theming`) *(core)* | Per-organization branding: logo, HSL colors, fonts and sanitised custom CSS. |
| [LearnDash migrator](migrator-learndash.md) (`mod.migrator-learndash`) | A complete migration from WordPress + LearnDash with ETL, staging and auditing. |
| [Hello World](hello-world.md) (`mod.hello-world`) | The reference template for [building new modules](../crear-un-modulo/index.md). |
