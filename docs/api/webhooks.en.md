# Webhooks

Didacta uses webhooks in both directions:

| | Who calls whom | What for | Where it is configured |
|---|---|---|---|
| **[Incoming](#incoming-webhooks)** | Stripe, Zoom, WooCommerce → Didacta | So the classroom learns about a payment, a class starting or an order | At the provider, pointing to your classroom's domain |
| **[Outgoing](#outgoing-webhooks)** | Didacta → your system | So your CRM, your shop or your n8n learns what happens in the classroom | **Administration → Webhooks** or `/api/v1/webhooks/endpoints` |

!!! info "A reference page"
    This page lists exact route names, headers, events and limits. For the "my shop lives elsewhere" use case, start with [Selling and integrating from outside](external-integrations.md).

---

## Incoming webhooks

### Common rules

- **The URL hangs off your classroom's domain** (`https://your-classroom.example.com/api/v1/…`). Didacta looks at the request's `Host` **before reading any secret** to know which classroom it is. That is why each classroom points its provider's webhook at **its own verified domain**; on an installation with several classrooms, each one has its own URL and its own secret.
- **Nothing is processed without a valid signature**: these are public routes (the provider sends no session) and authentication is the signature of the raw body with **that classroom's** secret.
- **The classroom that signs is the classroom it is processed for.** If a signed event names another classroom inside (through `metadata`, order or meeting ids), it is rejected with `403` and leaves a trace in the audit log.
- **Idempotent**: each event is stored before being processed, with its identifier as a unique key. A redelivery duplicates nothing, so the provider can retry safely.

### Stripe

There are two routes, one per module. Each verifies the `stripe-signature` header with its own secret (**Administration → Settings → Payments**, `/admin/settings?tab=payments`; if the classroom has none of its own, the instance one is used — see [Environment variables](../configuration/environment-variables.md)).

| Route | Module | Secret |
|---|---|---|
| `POST /api/v1/modules/billing/webhook` | Single course purchase ([Stripe Checkout](../modules/catalog/billing.md)) | "Webhook secret" |
| `POST /api/v1/modules/subscriptions/webhook` | Subscriptions and membership ([Subscriptions](../modules/catalog/subscriptions.md)) | "Subscriptions webhook secret" (optional: if left empty, the one above is used) |

Stripe events you must enable on each endpoint:

| Endpoint | Events |
|---|---|
| `…/billing/webhook` | `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`, `checkout.session.expired`, `charge.refunded` |
| `…/subscriptions/webhook` | `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`, `charge.refunded`, `checkout.session.completed` (membership sign-up), `checkout.session.expired` |

!!! tip "A single endpoint can serve both modules"
    The subscriptions route hands every verified event to subscriptions, membership and, last, single course purchase, and each module only processes its own. If you sell single courses **and** subscriptions with the same Stripe account, you can register **one** endpoint (`…/subscriptions/webhook`) with the **union** of the events in the table.

!!! warning "Without these events, you get charged but nothing is delivered"
    Access is granted on `checkout.session.completed` (or `async_payment_succeeded` for deferred payment methods) and removed on `charge.refunded` or `customer.subscription.deleted`. If you do not enable them, the payment lands in Stripe and the student does not get the course, or a refund does not remove access.

Useful responses when debugging:

| HTTP | Meaning |
|---|---|
| `200` | Received and processed (or archived with no effect if the event is not for this module) |
| `401` | `stripe-signature` is missing, the signature is invalid, or Stripe is not configured for this classroom |
| `404` | The request's domain does not match any classroom |
| `403` | The signed event names another classroom |

Stripe expects a fast `2xx` and, if it does not get one, retries for days. Because Didacta stores the event before processing it, a retry duplicates nothing.

!!! note "Platform mode"
    If the installation's operator turns on the platform payments mode (variable `PAYMENTS_PLATFORM_STRIPE_SECRET_KEY`), the Stripe webhook is **a single one** for the whole installation (`POST /api/v1/modules/payouts/webhook`, with the operator's secrets): classrooms configure nothing, and the classroom is decided by the platform's object registry, not by the domain.

### Zoom

`POST /api/v1/webhooks/zoom` — in your Zoom app, use **your classroom's** URL as the event endpoint: `https://your-classroom.example.com/api/v1/webhooks/zoom`.

1. In **Administration → Zoom**, save the **Webhook Secret Token** of your Zoom app. It is **per classroom** (stored encrypted next to the Server-to-Server credentials). Save it **before** pressing "Validate" in Zoom: the validation is signed with that token and, without it, Didacta answers `401`.
2. In Zoom, subscribe the app to the meeting and recording events.

| | |
|---|---|
| **Signature** | `x-zm-signature: v0=<hmac-sha256 hex>` over `v0:<x-zm-request-timestamp>:<raw body>`; the timestamp may not drift more than 5 minutes |
| **Handshake** | Answers the `endpoint.url_validation` event with `plainToken` and `encryptedToken` |
| **Events that change something** | `meeting.started` (class goes live), `meeting.ended` (class ended; triggers the post-class survey if enabled) and `recording.completed` (attaches the recording) |
| **Any other event** | `200` with `{"result":"IGNORED"}`, so Zoom does not retry |
| **Idempotency** | By a key derived from the notice's content (Zoom does not send an `event_id`) |

Several classrooms of the same owner may share **one** Zoom app and one URL **if they use exactly the same Secret Token**; in any other case a notice about another classroom's meeting is rejected with `403`.

The delivery log is in **Administration → Webhooks → Zoom tab**.

### WooCommerce

`POST /api/v1/modules/payment-connections/woo-webhook?tenant=<classroom-slug>` — called by your WooCommerce shop. Unlike the ones above, the classroom travels in the query (`?tenant=`), because WooCommerce knows nothing about multi-classroom setups.

| | |
|---|---|
| **Signature** | `x-wc-webhook-signature`: HMAC-SHA256 of the raw body with the webhook secret, in **base64** (not hex) and with no prefix |
| **Topics** | `order.created`, `order.updated`, `order.deleted`. Other topics: `200` ignored, so WooCommerce does not disable the webhook |
| **Verification ping** | The unsigned test `POST` WooCommerce sends when saving it gets `200` and is not processed |
| **Effect** | **Mirrors the order** onto the person's record; it does not grant access |
| **Order with no email** | `200` with `skipped`, no retries |

See [Payment connections](../modules/catalog/payment-connections.md).

### Managed video (Bunny Stream)

`POST /api/v1/video/bunny/webhook` is called by the managed video service when a video's status changes (signature `x-bunnystream-signature`). The installation's operator configures it, not the classroom.

---

## Outgoing webhooks

Outgoing webhooks notify your system —over HTTPS, with a JSON `POST`— every time a domain event happens in the classroom: an enrolment, a payment, a renewal, a finished course.

### Configuring them

In **Administration → Webhooks → Outgoing** (`/admin/webhooks`), or through the API with an administrator role (`tenant_admin` or `super_admin`):

| Method | Route | What for |
|---|---|---|
| `GET` | `/api/v1/webhooks/info` | Active tier, effective limits and the catalog of subscribable events |
| `GET` | `/api/v1/webhooks/endpoints[/:id]` | List / detail (the secret comes back masked) |
| `POST` | `/api/v1/webhooks/endpoints` | Creates an endpoint |
| `PUT` | `/api/v1/webhooks/endpoints/:id` | Edits; sending `secret` rotates it |
| `DELETE` | `/api/v1/webhooks/endpoints/:id` | Deletes (`204`, idempotent) |

```bash
curl -X POST https://your-classroom.example.com/api/v1/webhooks/endpoints \
  -H "Authorization: Bearer <accessToken>" \
  -H "Content-Type: application/json" \
  -d '{
        "url": "https://crm.example.com/hooks/didacta",
        "eventTypes": ["learning.enrollment.created", "billing.order.*"]
      }'
```

- `url`: **HTTPS** is mandatory (`http://` is only accepted for `localhost` and `127.0.0.1`, for development). The same URL cannot be repeated in a classroom (`409`).
- `eventTypes`: at least one, from the list below. `["*"]` receives **all** of them; a wildcard prefix (`billing.*`, `learning.*`) receives everything under that prefix. Names are not validated against the catalog: one that does not exist gives no error, but will never arrive.
- `secret`: optional (16–256 characters). If you do not send one, Didacta generates one (`whsec_…`). **It is returned in plaintext only once**, on creation or rotation; afterwards it only shows masked.
- `active`: `false` pauses the endpoint without deleting it.

### Event catalog

These are **all** the events you can subscribe to; each one has a real `publish()` behind it. (Modules emit more events among themselves, but only these are forwarded outside; see [Events and hooks](../modules/building-a-module/events-and-hooks.md).) Inside `data`, besides what is listed, come any fields a module adds in the future: tolerate new fields.

| Event | When | Main `data` fields |
|---|---|---|
| `learning.enrollment.created` | Someone becomes enrolled, by whatever route | `enrollmentId`, `userId`, `courseId`, `source` |
| `learning.enrollment.paused` | Enrolment suspended (e.g. for non-payment): keeps progress, loses access | `courseId`, `userId` |
| `learning.enrollment.resumed` | Access recovered | `courseId`, `userId` |
| `learning.enrollment.cancelled` | Enrolment ended (refund, cancellation, leaving a group) | `courseId`, `userId` (or `enrollmentId`) |
| `learning.course.completed` | Finishes the course | `enrollmentId`, `courseId`, `userId` |
| `community.post.created` | New post | `postId`, `authorId`, `courseId` |
| `community.comment.created` | New comment | `commentId`, `postId`, `authorId`, `parentCommentId` |
| `billing.order.created` | A single-course purchase was started (not paid yet) | `orderId`, `productId`, `courseId`, `userId`, `stripeSessionId`, `amount`, `currency` |
| `billing.order.completed` | Purchase paid | `orderId`, `productId`, `courseId`, `userId`, `userCreated`, `amountPaid`, `currency`, `customerEmail`, `attribution` |
| `billing.order.failed` | The payment failed, or the deferred one was never collected | `orderId`, `productId`, `courseId`, `userId` |
| `billing.order.refunded` | Purchase refunded | `orderId`, `productId`, `courseId`, `userId`, `amountRefunded` |
| `subscriptions.subscription.created` | A course subscription was started | `subscriptionId`, `courseId`, `userId`, `stripeSessionId`, `priceId`, `interval` |
| `subscriptions.subscription.activated` | Subscription active (sign-up or recovery after non-payment) | `subscriptionId`, `courseId`, `planId`, `userId`, `currentPeriodEnd`, `recovery` |
| `subscriptions.subscription.past_due` | A charge failed and the student is still in (grace period): the window to recover it | `subscriptionId`, `courseId`, `planId`, `userId`, `gracePeriodEndsAt` |
| `subscriptions.subscription.unpaid` | The grace period ran out without collecting | `subscriptionId`, `courseId`, `planId`, `userId` |
| `subscriptions.subscription.canceled` | Subscription ended | `subscriptionId`, `courseId`, `planId`, `userId`, `immediate` |
| `subscriptions.invoice.paid` | **Renewal collected**: the only event that tells "still paying" from "signed up once" | `subscriptionId`, `stripeInvoiceId`, `amount`, `currency` |
| `subscriptions.invoice.payment_failed` | The instalment's charge failed | `subscriptionId`, `stripeInvoiceId`, `attemptCount`, `nextPaymentAttempt` |
| `subscriptions.invoice.refunded` | Instalment refunded (published **without a person**: `learner: null`) | `subscriptionId`, `stripeInvoiceId`, `amountRefunded`, `currency` |
| `subscriptions.membership.activated` | A membership sign-up | `subscriptionId`, `planId`, `userId`, `userCreated`, `stripeSubscriptionId`, `attribution` |
| `fundae.group.closed` | A Fundae group is closed | `groupId`, `companyId`, `creditoConsumidoCents` |
| `assessments.attempt.submitted` | An exam attempt is submitted | `attemptId`, `quizId`, `userId`, `enrollmentId`, `lessonId`, `scoreEarned`, `scoreMax`, `scorePercent`, `passed`, `needsReview` |

!!! note "Events that are published but not forwarded"
    Among others, `billing.order.partially_refunded`, `subscriptions.invoice.partially_refunded` and `subscriptions.subscription.paused` are **not** in this version's outgoing catalog. If you need them, use `learning.enrollment.*` to know whether access changed, or ask for them to be added.

### The body

```json
{
  "event": "learning.enrollment.created",
  "data": {
    "enrollmentId": "…",
    "userId": "…",
    "courseId": "…",
    "source": "API"
  },
  "occurredAt": "2026-11-16T17:24:05.412Z",
  "tenantId": "…",
  "deliveryId": "wd_9f0c…",
  "learner": {
    "id": "…",
    "email": "ana@example.com",
    "name": "Ana Ruiz",
    "externalSource": null,
    "externalId": null
  }
}
```

| Field | What it is |
|---|---|
| `event` | The event name |
| `data` | The event's data (table above) |
| `occurredAt` | When the event **happened** (ISO 8601), not when it was sent: after a redelivery they can differ by hours |
| `tenantId` | The classroom |
| `deliveryId` | The delivery's identifier, **stable** across retries and redeliveries of the same event: your deduplication key |
| `learner` | The person the event is about: `id`, `email`, `name`, `externalSource` and `externalId` (the key under which the source system knows that same person, in migrated classrooms). **`null`** if the event has no person behind it (`subscriptions.invoice.refunded`) |
| `attempt` | Enterprise only: the queue's attempt number |

`data` only carries a `userId`, and there is no endpoint that translates a `userId` into a person: that is why `learner` arrives already resolved. The subject is the event's own `userId` and, if it has none, the actor that caused it.

### Headers

| Header | Value |
|---|---|
| `Content-Type` | `application/json` |
| `X-Didacta-Event` | The event name |
| `X-Didacta-Delivery` | The `deliveryId`. **Deduplicate on this header** |
| `X-Didacta-Attempt` | Attempt number |
| `X-Didacta-Tenant` | The `tenantId` |
| `X-Didacta-Signature` | `sha256=<hex>`. **Enterprise only** (see below) |

### Community, Enterprise and Cloud

Behaviour depends on whether the installation has the **`feat:api.webhooks.high_throughput`** capability (see [Capabilities](../enterprise/capabilities.md)). It is evaluated on every send: changing licence takes effect without a restart.

| | Community | Enterprise (`feat:api.webhooks.high_throughput`) |
|---|---|---|
| Endpoints per classroom | 1 (`WEBHOOKS_COMMUNITY_MAX_ENDPOINTS`) | 20 (`WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS`) |
| Event types per endpoint | 3 (`WEBHOOKS_COMMUNITY_MAX_EVENTS`); `["*"]` counts as one and receives the whole catalog | Unlimited |
| Delivery | **Direct and synchronous**, within the event's dispatch | Dedicated queue (`didacta.webhooks.outbound`, BullMQ + Redis) |
| Retries | 1 immediate retry (after 250 ms), 2 attempts in total | 5 retries with growing waits: **30 s, 2 min, 8 min, 30 min, 2 h** (6 attempts in total) |
| Signature | **Unsigned** | **HMAC-SHA256** in `X-Didacta-Signature` |
| When retries run out | The event is lost (only a warning in the server log remains) | It goes to the **dead-letter**: retry or dismiss it from the panel or the API |
| Metrics | — | Prometheus: `webhooks_outbound_total`, `webhooks_outbound_duration_seconds`, `webhooks_outbound_attempts`, `webhooks_outbound_dead_letter_total`, `webhooks_outbound_queue_depth` |

**Didacta Cloud**: the `cloud` edition has its own licence, and the classroom applies the Enterprise column **if that licence includes** `feat:api.webhooks.high_throughput`; otherwise, the Community one. Limits depend not on the edition's name but on that capability.

In every case:

- Each `POST` has a **5 s** timeout (`WEBHOOKS_TIMEOUT_MS`). Any non-`2xx` response counts as a failure. **Answer `2xx` as soon as you receive**, then process.
- Notices are **best-effort**: a down endpoint never slows the business process nor the other endpoints.
- A delivery can arrive **more than once**. Deduplicate on `X-Didacta-Delivery`.

!!! warning "In Community the `POST` is not signed"
    If you need to make sure the notice comes from your classroom, put a **secret token in the endpoint's own URL** (`https://crm.example.com/hooks/didacta?token=…`) and check it, or filter by your installation's IP.

### Verifying the signature (Enterprise)

The signature is `sha256=` + the hexadecimal HMAC-SHA256 of the **exact body** you receive (the raw bytes, before parsing the JSON), with the endpoint's `secret` as the key.

```js
import { createHmac, timingSafeEqual } from 'node:crypto';

function isFromDidacta(rawBody, signatureHeader, secret) {
  const expected = 'sha256=' + createHmac('sha256', secret).update(rawBody, 'utf8').digest('hex');
  const a = Buffer.from(signatureHeader ?? '');
  const b = Buffer.from(expected);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

!!! warning "Exception: the no-Redis fallback"
    If Didacta cannot enqueue the delivery (for example, Redis is down at that instant), it sends it directly, as in Community, so the event is not lost: **that delivery carries no `X-Didacta-Signature`**. If your receiver rejects anything unsigned you will lose those notices; instead, accept unsigned ones only if they carry your token in the URL.

### Dead-letter (Enterprise)

Deliveries that exhaust their retries are kept for review. They are in **Administration → Webhooks** and through the API (administrator role; without a licence they answer `402`):

| Method | Route | What for |
|---|---|---|
| `GET` | `/api/v1/admin/webhooks/dead-letter?limit=` | List (1 to 200; 50 by default) with `eventType`, `payload`, `lastError`, `attempts` |
| `POST` | `/api/v1/admin/webhooks/dead-letter/:id/retry` | Re-enqueues the delivery (`202`) and deletes the row if re-enqueuing worked |
| `DELETE` | `/api/v1/admin/webhooks/dead-letter/:id` | Dismisses it (`204`) |

!!! note "A re-enqueued delivery arrives poorer"
    It is rebuilt from the event and its `data`: it arrives with `learner: null` and a `deliveryId` ending in `-legacy`, not the original. If your receiver deduplicates on `deliveryId`, it will not mistake it for a duplicate.

### Environment variables

| Variable | Default | What it controls |
|---|---|---|
| `WEBHOOKS_TIMEOUT_MS` | `5000` | Timeout of the `POST` to an endpoint |
| `WEBHOOKS_COMMUNITY_MAX_ENDPOINTS` | `1` | Endpoints per classroom in Community |
| `WEBHOOKS_COMMUNITY_MAX_EVENTS` | `3` | Event types per endpoint in Community |
| `WEBHOOKS_ENTERPRISE_MAX_ENDPOINTS` | `20` | Endpoints per classroom with the capability |

The Enterprise queue requires `REDIS_URL`.

## See also

- [Selling and integrating from outside](external-integrations.md) — which events to use for an external shop.
- [Events and hooks](../modules/building-a-module/events-and-hooks.md) — the complete catalog of domain events each module declares.
- [Conventions](conventions.md) — errors, multi-tenancy and request limits.
- [Enterprise capabilities](../enterprise/capabilities.md).
