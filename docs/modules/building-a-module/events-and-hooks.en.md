# Events and hooks

Modules never call each other: they communicate through **domain events** (asynchronous, with a transactional outbox) and **hooks** (synchronous extension points). Everything a module emits or consumes is declared in the manifest.

## Domain events

### The contract

```ts
interface DomainEvent<TPayload = unknown> {
  name: string;          // 'courses.course.published'
  version: number;
  data: TPayload;
  metadata: {
    tenantId: string;
    userId?: string;     // the ACTOR, which is not always the event's subject
    timestamp: string;
    traceId: string;
    idempotencyKey: string;
  };
}

interface EventBus {
  publish<T>(event: DomainEvent<T>): Promise<void>;
  subscribe<T>(eventName: string, handler: (e: DomainEvent<T>) => Promise<void> | void): () => void;
}
```

Naming convention: `<scope>.<entity>.<action>` (`courses.course.published`, `subscriptions.subscription.canceled`, `learning.course.completed`). The scope is usually the module's slug, but not always: `mod.zoom-live` emits `zoom.session.*`, `mod.member-registration` emits `member_registration.*` and `mod.payment-connections` emits `payment_connections.*`.

### Guaranteed delivery: transactional outbox

`publish()` does not fire the event directly: it persists it in the `outbox_event` table (deduplicating on `(tenantId, idempotencyKey)`) and a worker dispatches it through BullMQ. Without Redis, dispatch is immediate and in-process; a recovery worker re-enqueues pending rows. Result: an event published inside a transaction is **not lost** even if the process dies.

Each `outbox_event` row is in one of four states:

| `processed_at` | `last_error` | Meaning |
|---|---|---|
| set | empty | Delivered to at least one handler, all succeeded |
| set | `NO_HANDLER: <event>` | **It reached nobody.** A labelled terminal state, not a success: if a subscriber shows up within the next 24 hours, it is redelivered |
| empty | empty | Pending: enqueued, not yet dispatched |
| empty | an error | A handler failed; it is retried |

An event with no subscribers is legitimate (a bus may fan out to zero), but note that **"delivered" and "nobody was there" are not the same thing**.

### The idempotency key

Compute the `idempotencyKey` with the kernel helper, not by hand:

```ts
import { eventIdempotencyKey } from '@didacta/core-kernel';

idempotencyKey: eventIdempotencyKey('community.post.created', data /*, uniqueSuffix? */),
```

It returns `<name>:<payload digest>:<suffix>` and **never exceeds 256 characters**. Building it by hand with the whole payload inside (`${name}:${JSON.stringify(data)}:${Date.now()}`) blows up the table's unique index once the payload grows: the business action is already done and the event —with the notices, webhooks and enrolments hanging off it— does not exist.

- The default `uniqueSuffix` is `Date.now()`. If the event has a **natural identifier for the occurrence** (the order, the attempt), pass it as the suffix: the key is then truly stable and a retry does not publish twice.

!!! warning "Choose the suffix carefully"
    Deduplication by `idempotencyKey` is **forever**. If an entity can legitimately re-emit the same state (a subscription that activates, cancels and reactivates months later), the suffix has to be **per occurrence** (`randomUUID()`, or a counter), not stable — or the second event will be silently swallowed.

### Subscribing

Subscriptions are made in `onRegister` (once per process, no tenant) or in a host *bridge* (`apps/api/src/modules/<slug>/*.bridge.ts`, which is where one module reacts to another without importing it):

```ts
// apps/api/src/modules/access-groups/access-groups-courses.bridge.ts (pattern)
eventBus.subscribe('courses.course.published', async (event) => {
  // 1) Is my module active for event.metadata.tenantId? If not, return.
  // 2) React (ALWAYS filtering by tenantId).
});
```

Remember: disabling a module does **not** unsubscribe its handlers — check the state inside the handler. All the handlers of one event run even if one fails, and always inside the event's tenant context. If any of them throws, the whole dispatch is marked as failed and retried —**including the handlers that had already finished fine**— so **every handler must be idempotent**.

### Declaration in the manifest

Every published event goes in `eventsEmitted`; every subscribed event, in `eventsConsumed`. Emitting an undeclared event is an anti-pattern that review rejects.

!!! note "Third-party modules (sandbox)"
    A marketplace module's manifest also carries `eventsEmitted` and `eventsConsumed`, but the host does not yet expose an event client (`ctx.events`) to modules running in the sandbox: for now, declaring events there has no effect. Third-party modules integrate through the [core's public APIs](../third-party-modules.md).

## Domain event catalog

This is what each in-tree module declares in its manifest, and it matches what the code publishes: `scripts/module-doctor.ts` (CI) fails if a module declares an event that appears in no `publish()`, or if a module's code mentions one of its own events that is not declared. It is a static check: it confirms the event exists in the code, not that every path runs it.

| Module | Emits | Consumes |
|---|---|---|
| `mod.courses` | `courses.course.created`, `.updated`, `.published`, `.archived`, `.unarchived`; `courses.module.created`, `.deleted`; `courses.lesson.created`, `.updated`, `.moved`, `.moved-to-module`, `.deleted`; `courses.lessons.reordered`, `courses.modules.reordered` | — |
| `mod.learning` | `learning.enrollment.created`, `.paused`, `.resumed`, `.cancelled`; `learning.progress.updated`; `learning.course.completed`; `learning.invitation.created`; `scorm.attempt.completed` | `courses.course.archived` |
| `mod.assessments` | `assessments.quiz.published`; `assessments.attempt.started`, `.submitted`, `.pending_review`, `.graded`, `.passed`, `.failed` | — |
| `mod.certificates` | `certificates.issued`, `certificates.revoked` | `learning.course.completed` |
| `mod.access-groups` | — | `courses.course.published`, `payment_connections.user_tier.changed`, `subscriptions.membership.activated`, `subscriptions.subscription.activated`, `.canceled`, `.unpaid` |
| `mod.billing` | `billing.order.created`, `.completed`, `.failed`, `.refunded`, `.partially_refunded` | — |
| `mod.subscriptions` | `subscriptions.subscription.created`, `.activated`, `.past_due`, `.unpaid`, `.canceled`, `.paused`; `subscriptions.invoice.paid`, `.payment_failed`, `.refunded`, `.partially_refunded`; `subscriptions.membership.activated`; `subscriptions.retention.paused`, `.discounted`, `.canceled` | — |
| `mod.payment-connections` | `payment_connections.user_tier.changed` | — |
| `mod.referrals` | `referrals.referral.attributed`; `referrals.commission.created`, `.approved`, `.revoked`; `referrals.payout.recorded`; `referrals.credit.returned` | `subscriptions.membership.activated`, `subscriptions.invoice.paid`, `.refunded`, `.partially_refunded`, `subscriptions.subscription.canceled`, `billing.order.completed`, `.refunded`, `.partially_refunded` |
| `mod.community` | `community.post.created`, `community.comment.created`, `community.reaction.added`; `community.post.hidden`, `.unhidden`; `community.comment.hidden`, `.unhidden` | — |
| `mod.messaging` | `messaging.conversation.created`, `messaging.message.sent` | — |
| `mod.gamification` | `gamification.points.awarded`, `.revoked`; `gamification.level.changed`; `gamification.challenge.submitted`, `.reviewed`; `gamification.perk.requested`, `.handled`; `gamification.badge.awarded`; `gamification.reward.granted`; `gamification.content.unlocked` | `community.post.created`, `community.comment.created`, `community.post.hidden`, `.unhidden`, `community.comment.hidden`, `.unhidden`, `resources.resource.created`, `.deleted`, `learning.course.completed`, `learning.progress.updated`, `referrals.referral.attributed`, `gamification.level.changed`, `gamification.points.awarded` |
| `mod.resources` | `resources.resource.created`, `.deleted`; `resources.collection.created` | — |
| `mod.surveys` | `surveys.survey.created`, `surveys.response.submitted` | `zoom.session.ended` |
| `mod.zoom-live` | `zoom.session.created`, `.updated`, `.cancelled`, `.started`, `.ended`, `.recording_ready`, `.attendance_synced`; `zoom.session.registration.created`, `.cancelled` | — |
| `mod.member-registration` | `member_registration.request.created`, `.approved`, `.rejected` | — |
| `mod.retention` | `retention.message.sent` | `learning.course.completed`, `community.post.created`, `gamification.level.changed` |
| `mod.soporte` | `soporte.ticket.created`, `soporte.ticket.status_changed` | — |
| `mod.fundae` | `fundae.action.*`, `fundae.company.*`, `fundae.rlpt.notice.*`, `fundae.group.created`, `.updated`, `.started`, `.closed`, `.cancelled`, `fundae.cost.*`, `fundae.group.participant.*`, `fundae.group.start-xml.generated`, `.end-xml.generated`, `.completion-computed`, `.audit-zip.generated`, `fundae.export.generated`, `fundae.zip.generated`, `fundae.block.*`, `fundae.inspector.granted`, `.revoked` | — |
| `mod.ai-tutor` | `ai-tutor.course.indexed`, `.index-failed`; `ai-tutor.chat.message-received`; `ai-tutor.answer.reviewed`, `.corrected` | `courses.course.published`, `.unpublished`, `courses.lesson.created`, `.updated`, `.deleted` |
| `mod.ai-grader` | `ai-grader.suggestion.generated`, `.failed` | — |
| `mod.ai-content` | `ai-content.draft.generated`, `.published`, `.rejected` | — |
| `mod.theming` | `theming.logo.uploaded`, `theming.favicon.uploaded` | — |
| `mod.wp-sso` | — | — |
| `mod.hello-world` | `hello-world.greeting.requested` | — |
| `mod.payouts`, `mod.migrator-learndash` | — | — |

**What is deliberately not there.** `mod.wp-sso` emits no events on the bus: the token exchange leaves its trace in the **audit log** (`sso.wp.signin.success`, `sso.wp.user.provisioned`, `sso.wp.account.linked`, `sso.wp.callback.rejected`). There is no `subscriptions.membership.revoked` either: a membership losing access travels in `subscriptions.subscription.canceled` (immediate cancellation), `.unpaid` and `.paused`, with `planId` in the payload. And a tutor turn is **one** event, `ai-tutor.chat.message-received`; there is no separate `message-sent`.

**New payloads.** `certificates.revoked` carries `{ userId, courseId, reason, revokedCount }` and is published when a course's certificates are revoked (today, when the purchase is refunded); `ai-grader.suggestion.generated` carries `{ suggestionId, attemptId, answerId, questionId, proposedScore, provider, model }` and `ai-grader.suggestion.failed` carries `{ attemptId, answerId, questionId, reason }`, with `reason` = `provider` or `unparseable`. `learning.enrollment.paused` and `.resumed` are published only if an enrollment actually changed state. When you add an event, **always** declare in the manifest what you publish.

An event's name and shape are defined by the module that emits it; look at its `data` in the service that calls `publish()`.

## Hooks

Hooks are **synchronous inside another module's flow**: they let a module step into an operation that belongs to someone else without the owner knowing about the guest.

```ts
interface HookRegistry {
  register<TInput>(hookName: string, handler: (ctx: HookContext<TInput>) => Promise<void> | void): void;
  run<TInput>(hookName: string, ctx: HookContext<TInput>): Promise<void>;
}

interface HookContext<TInput> {
  tenantId: string;
  input: TInput;                    // the object the owner shares and handlers may mutate
  metadata: Record<string, unknown>;
}
```

### The hooks that exist today

| Hook | Exposed by | When it runs | `input` |
|---|---|---|---|
| `courses.publish.validate` | `mod.courses` | When publishing a course, right before its status changes (and after checking it has lessons) | `{ courseId, reasons: string[] }`; `metadata.actorId` |

It is the **only** hook any module exposes, and in this version **no module registers a handler** for it: it is an open extension point, not a mechanism in use.

### Exposing a hook (the owner)

`mod.courses` declares it in its manifest:

```ts
hooksExposed: [
  {
    name: 'courses.publish.validate',
    description: 'Lets other modules add validations before a course is published.',
    async: true,
  },
],
```

And fires it in its service:

```ts
const reasons: string[] = [];
await this.ctx.hookRegistry.run('courses.publish.validate', {
  tenantId,
  input: { courseId, reasons },
  metadata: { actorId },
});
if (reasons.length > 0) throw new PublishValidationError(reasons);
// if reasons ended up with entries, publication is blocked
```

### Registering for a hook (the guest)

A module registers its handler in `onRegister`, without the owner knowing it exists (pattern; nobody uses it for this hook today):

```ts
ctx.hookRegistry.register<{ courseId: string; reasons: string[] }>(
  'courses.publish.validate',
  async ({ input }) => {
    if (!meetsMyRequirement(input.courseId)) input.reasons.push('My requirement is missing');
  },
);
```

Registry rules:

- Handlers run **sequentially**, in registration order, and report their result by **mutating the shared `input`** (the `reasons[]` pattern for vetoes).
- A handler that **throws** aborts the owner's operation with that error: to veto, add a reason to `reasons`, do not throw.
- The registry is **in-memory and per process**, with no tenant distinction: as with events, check inside the handler that your module is active for `ctx.tenantId`.
- A hook with no handlers does nothing.

## Events and hooks vs. public services

When you need **data** from another module synchronously, the preferred route is not a hook but its **public service** injected by the host (route A of ADR-016) — or, as a last resort, reading its tables directly with the dependency declared (route B, read-only). See [Database](database.md#reading-another-modules-data-route-b-of-adr-016).

## Outgoing webhooks

A domain event **does not leave the system** automatically. Only the events listed in the host's **webhook catalog** (`KNOWN_EVENT_TYPES` in `apps/api/src/webhooks/webhooks.types.ts`) are forwarded to the outgoing webhooks administrators configure: `learning.enrollment.*`, `learning.course.completed`, `community.post.created`, `community.comment.created`, the orders (`billing.order.*`, not all of them), the subscription and membership lifecycle (`subscriptions.*`, not all of them), `fundae.group.closed` and `assessments.attempt.submitted`. The full list, with its payloads, is in [API → Webhooks](../../api/webhooks.md#event-catalog).

If your event should be subscribable from outside, propose adding it to the catalog. That list is **binding twice over** —the bridge only subscribes to what is in it and the UI only offers that—, so before adding an entry check that a real `publish()` stands behind it: the manifest declares what the module commits to emit and `module-doctor` checks the event exists in the code, but it does not prove every path publishes it (there was once a declared `subscriptions.membership.revoked` that nobody published, and it was removed). Check the `publish()`.
