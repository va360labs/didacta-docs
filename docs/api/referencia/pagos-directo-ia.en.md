# Reference — Payments, virtual classroom and AI

Billing, subscriptions, payment connections, Zoom and the three AI modules. Every route hangs off `/api/v1`.

## One-off payment (billing) — `/modules/billing`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/billing/offer/:courseId` | Bearer | The course's offer (`forSale`, price, discount). Tolerant: with no Stripe configured it returns `forSale: false`, never a 500. |
| POST | `/modules/billing/checkout/:courseId` | Bearer | Creates the Stripe Checkout Session and returns the URL. 409 if the course is not published or you already have access. |
| GET · POST · PATCH · DELETE | `/modules/billing/products[/:id]` | admin | Products (course ↔ Stripe price): creation with an existing `stripePriceId` **or** with `amountCents` (which creates the Product+Price), activation, deletion (historical orders are untouched). |
| GET | `/modules/billing/public/catalog` · `public/offer/:courseId` | Public (tenant by domain) | The public sales catalog · the public offer. Both degrade to empty with no Stripe. The offer **does not filter on existence**: a non-existent or unpublished course returns 200 `{ forSale: false, options: [] }`, never a 404. |
| POST | `/modules/billing/public/checkout/:courseId` | Public | **Anonymous checkout**: the visitor buys with no account. Body `{ optionId?, email? }` — the `email` only **pre-fills** the Stripe form; the account is ALWAYS created with the email confirmed at payment. 404 non-existent course (or a `:courseId` that is not a UUID) · 409 not published · 503 no payment gateway. |
| POST | `/modules/billing/webhook` | Stripe signature (`stripe-signature`) | Receives `checkout.session.completed/expired`, `charge.refunded`. Idempotent by `stripe_event_id`. |

## Subscriptions — `/modules/subscriptions`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/modules/subscriptions/checkout/:courseId` | Bearer | Checkout with `mode=subscription`: `{ stripePriceId }` (recurring). |
| GET | `/modules/subscriptions/me` | Bearer | My subscriptions (cancelled ones included, with plan and course). Returns `[]` if the module is not configured. |
| GET | `/modules/subscriptions/me/:id/invoices` | Bearer (owner) | Invoices with Stripe's hosted URL. |
| POST | `/modules/subscriptions/me/:id/cancel` | Bearer (owner) | Cancels: `{ immediate? }` (at the end of the period by default). |
| POST | `/modules/subscriptions/me/membership/pay-now` | Bearer | Ends the membership trial and charges right away. |
| POST | `/modules/subscriptions/webhook` | Stripe signature (its own secret) | `customer.subscription.*`, `invoice.*`; idempotent membership fulfillment. |
| POST | `/modules/subscriptions/admin/grace-expiration/run-now` | super_admin | Forces the expiry of grace periods (QA) → 202. |

Common errors: `SUBSCRIPTIONS_PRICE_NOT_RECURRING` 422 · `SUBSCRIPTIONS_ALREADY_ACTIVE` 409 · `*_STRIPE_CONFIG_MISSING` 503 · `*_STRIPE_API_ERROR` 502.

## Payment connections — `/modules/payment-connections` (super_admin)

**Connections** (read-only over the accounts):

| Method | Route | What it does |
|---|---|---|
| POST · GET | `…/connections` | Connects an account (Stripe with a restricted `rk_`/`sk_` key, PayPal or WooCommerce; credentials encrypted) · lists them without keys. |
| POST | `…/connections/:id/verify` | Re-validates the credentials. |
| GET | `…/connections/:id/reconcile` | Active subscribers split into `matched` / `unmatched` against the users, by email. |
| POST | `…/connections/:id/invite` | Bulk-invites those who are not registered (up to 200 emails; a per-email result, it never fails as a whole). |
| DELETE | `…/connections/:id` | Disconnects and deletes the encrypted key. |

**Tiers**: `GET/POST/PATCH/DELETE …/tiers/catalog[/:id]` (the catalog) · `GET …/user-tiers?userIds=` (the effective tier of up to 500 users) · `PUT …/user-tiers/:userId` (a manual tier) · `POST …/user-tiers/sync` (reconciles against the payments; publishes `payment_connections.user_tier.changed`).

**Order mirror (WooCommerce)**: `POST …/orders-mirror/sync?days=` · `GET …/orders-mirror/expiring?days=` · `GET/PUT …/orders-mirror/rules` (regex-based classification → `LIFETIME|SUBSCRIPTION|TIMED|ONE_OFF|INFRA`) · `PUT …/orders-mirror/webhook-secret` · `GET …/orders-mirror/webhook-status`.

**Subscription dashboard**: `POST …/subscriptions-dashboard/sync` · `GET …/subscriptions-dashboard[?filters]` · `GET …/summary` · `GET …/subscribers/:id/renewal-url` · `POST …/subscribers/:id/renewal-email` · `GET/PUT …/renewal-template` · `GET/PUT …/cancel-portal-url` · `POST …/daily/run-now`.

**User self-service** (Bearer): `GET …/me/subscription` (their external subscription, for `/cuenta`) · `POST …/me/billing-portal` (a Stripe Customer Portal session).

**Incoming webhook**: `POST /modules/payment-connections/woo-webhook?tenant=<slug>` — public, with an HMAC-SHA256 signature (`x-wc-webhook-signature`); it returns 200 to verification pings and to ignored topics so WooCommerce does not disable the webhook.

## Virtual classroom — `/modules/zoom-live`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/zoom-live/sessions[/:id]` | Bearer | List (filters by course/lesson/status/range) · detail. The `joinUrl` and the recording are **only** for registered attendees or staff. |
| POST | `/modules/zoom-live/sessions/:id/register` · `unregister` | Bearer | Registration (idempotent) · cancellation. |
| POST | `/modules/zoom-live/sessions/:id/join` | Bearer | Stamps the entry (the attendance proxy) and returns the `joinUrl`. |
| POST · PUT · DELETE | `/modules/zoom-live/sessions[/:id]` | staff | Creates (`topic`, `startTime`, `durationMinutes` 15-480, `hostEmail`, `timezone`, optional course/lesson) · edits · cancels (soft). |
| GET | `/modules/zoom-live/sessions/:id/registrations` | staff | The roster of registered attendees. |
| GET · POST · PUT | `/modules/zoom-live/sessions/:id/attendance[…]` | staff | The attendance report · sync against the Zoom API · a manual per-member override (`{ present: bool\|null }`). |
| POST | `/modules/zoom-live/test-credentials` | admin | A smoke test of the S2S credentials (`real` or `stub`). |
| GET | `/modules/zoom-live/webhook-events` | admin | The Zoom webhook log (QA). |
| GET | `/modules/zoom-live/sessions/:id/calendar.ics` · `calendar/google` · `outlook` · `office365` | **Public** | The calendar event (it never includes the `joinUrl`). |
| POST | `/webhooks/zoom` | Zoom HMAC signature | `meeting.started/ended`, `recording.completed` + the validation handshake. Idempotent by `event_id`. |

## AI

**Content generation** — `/modules/ai-content` (instructor+):

| Method | Route | What it does |
|---|---|---|
| POST | `/modules/ai-content/generate` | Generates a `SUMMARY\|FLASHCARDS\|QUIZ` draft from a lesson: `{ lessonId, courseId, type }`. |
| GET | `/modules/ai-content/drafts[/:id]` | Filtered list · detail. |
| PATCH | `/modules/ai-content/drafts/:id/content` · `publish` · `reject` | Edits the JSON before publishing · publishes · rejects with a reason. |

Errors: `AI_CONTENT_LESSON_TEXT_EMPTY` 422 · `AI_CONTENT_DRAFT_NOT_IN_DRAFT` 409 · `AI_CONTENT_PROVIDER_ERROR` **503**.

**Rubric-based grading** — `/modules/ai-grader` (instructor+):

| Method | Route | What it does |
|---|---|---|
| GET · PUT · DELETE | `/modules/ai-grader/questions/:questionId/rubric` | The rubric per question: `instructions` + 1-8 weighted criteria (the weights must add up to the question's points). |
| POST | `/modules/ai-grader/attempts/:attemptId/suggest` | Generates AI suggestions for the open-ended answers (`{ force? }` bypasses the cache). |
| GET | `/modules/ai-grader/attempts/:attemptId/suggestions` | The persisted suggestions (without calling the model). |
| POST | `/modules/ai-grader/suggestions/:id/apply` | Marks the suggestion as applied (for auditing). The real score is set in assessments. |

**Tutor** — `/modules/ai-tutor` and `/admin/ai-tutor`:

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/modules/ai-tutor/courses/:courseId/ask` | Bearer | Asks the tutor (RAG): `{ question, conversationId?, lessonId?, topK? }` → the answer + citations. Staff can try any course with no enrollment and no quota. |
| POST | `/admin/ai-tutor/courses/:courseId/index` · `reindex-all` | admin | Reindexes one course · every published course. |

Tutor errors: `AI_TUTOR_DAILY_QUESTION_QUOTA` / `AI_TUTOR_TOKEN_QUOTA_EXCEEDED` **429** · `AI_TUTOR_COURSE_NOT_INDEXED` 404 · `AI_TUTOR_COURSE_ACCESS_DENIED` 403 · `AI_PROVIDER_NOT_CONFIGURED` **424** · provider down 502.

The tutor's review panel (answers, validated knowledge, the monthly report) is in [Administration → AI providers](administracion.md#ai-adminai-admin).
