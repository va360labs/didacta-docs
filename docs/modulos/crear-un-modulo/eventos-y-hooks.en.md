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
    userId?: string;
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

Naming convention: `<module>.<entity>.<action>` (`courses.course.published`, `subscriptions.subscription.canceled`, `learning.course.completed`).

### Guaranteed delivery: transactional outbox

`publish()` does not fire the event directly: it persists it in the `outbox_event` table (deduplicating by `(tenantId, idempotencyKey)`) and a worker dispatches it over BullMQ. Without Redis, dispatch is immediate and in-process; a recovery worker requeues anything pending. The result: an event published inside a transaction **is not lost** even if the process crashes.

!!! warning "Choose the `idempotencyKey` carefully"
    Deduplication by `idempotencyKey` is **permanent**. If an entity can legitimately re-emit the same state (a subscription that is activated, cancelled and reactivated months later), use a **per-occurrence** key (`${name}:${id}:${randomUUID()}`), not a stable one — otherwise the second event is swallowed silently.

### Subscribing

Subscriptions happen in `onRegister` (once per process, with no tenant) or in a host *bridge*:

```ts
// apps/api/src/modules/access-groups/access-groups-courses.bridge.ts (the pattern)
ctx.eventBus.subscribe('courses.course.published', async (event) => {
  // 1) Is my module active for event.metadata.tenantId? If not, return.
  // 2) React (ALWAYS filtering by tenantId).
});
```

Remember: disabling a module does **not** unsubscribe its handlers — check the state inside the handler.

### Declaration in the manifest

Every published event goes in `eventsEmitted`; every subscribed event, in `eventsConsumed`. Emitting an undeclared event is an anti-pattern that review will reject.

## Hooks

Hooks are **synchronous, inside another module's flow**: they let a module take part in someone else's operation without the owner knowing the guest exists.

### Exposing a hook (the owner)

`mod.courses` declares in its manifest:

```ts
hooksExposed: [
  {
    name: 'courses.publish.validate',
    description: 'Permite añadir validaciones antes de publicar un curso.',
    async: true,
  },
],
```

And fires it from its service:

```ts
await this.ctx.hookRegistry.run('courses.publish.validate', {
  tenantId,
  input: { courseId, reasons: [] },
  metadata: {},
});
// if input.reasons ends up with entries, publishing is blocked
```

### Registering with a hook (the guest)

`mod.fundae` adds its validation without `mod.courses` knowing it exists:

```ts
ctx.hookRegistry.register<{ courseId: string; reasons: string[] }>(
  'courses.publish.validate',
  async ({ input }) => {
    if (!hasObjective(input.courseId)) input.reasons.push('Falta objetivo Fundae');
  },
);
```

Handlers run sequentially; they communicate their result by mutating the shared `input` (the `reasons[]` pattern for vetoes).

## Events and hooks vs. public services

When you need **data** from another module synchronously, the preferred route is not a hook but its **public service**, injected by the host (route A of ADR-016) — or, as a last resort, reading its tables directly with the dependency declared (route B, read-only). See [Database](base-de-datos.md#reading-another-modules-data-route-b-of-adr-016).

## Outgoing webhooks

Domain events flagged in the webhook catalog (`learning.enrollment.created`, `community.post.created`, `billing.subscription.*`…) are also forwarded to the **outgoing webhooks** administrators configure. If your event ought to be subscribable from outside, propose adding it to the catalog. See [API → Conventions](../../api/convenciones.md#outgoing-webhooks).
