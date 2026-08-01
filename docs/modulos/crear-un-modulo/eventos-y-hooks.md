# Eventos y hooks

Los módulos no se llaman entre sí: se comunican por **eventos de dominio** (asíncronos, con outbox transaccional) y **hooks** (síncronos, puntos de extensión). Todo lo que un módulo emite o consume se declara en el manifest.

## Eventos de dominio

### El contrato

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

Convención de nombres: `<modulo>.<entidad>.<accion>` (`courses.course.published`, `subscriptions.subscription.canceled`, `learning.course.completed`).

### Entrega garantizada: transactional outbox

`publish()` no lanza el evento directamente: lo persiste en la tabla `outbox_event` (deduplicando por `(tenantId, idempotencyKey)`) y un worker lo despacha vía BullMQ. Sin Redis, el despacho es in-process inmediato; un worker de recuperación reencola los pendientes. Resultado: un evento publicado dentro de una transacción **no se pierde** aunque el proceso caiga.

!!! warning "Elige bien la `idempotencyKey`"
    La deduplicación por `idempotencyKey` es **para siempre**. Si una entidad puede reemitir legítimamente el mismo estado (una suscripción que se activa, se cancela y se reactiva meses después), usa una clave **por ocurrencia** (`${name}:${id}:${randomUUID()}`), no una clave estable — o el segundo evento se tragará en silencio.

### Suscribirse

Las suscripciones se hacen en `onRegister` (una vez por proceso, sin tenant) o en un *bridge* del host:

```ts
// apps/api/src/modules/access-groups/access-groups-courses.bridge.ts (patrón)
ctx.eventBus.subscribe('courses.course.published', async (event) => {
  // 1) ¿Está mi módulo activo para event.metadata.tenantId? Si no, return.
  // 2) Reaccionar (filtrando SIEMPRE por tenantId).
});
```

Recuerda: desactivar un módulo **no** desuscribe sus handlers — comprueba el estado dentro del handler.

### Declaración en el manifest

Todo evento publicado va en `eventsEmitted`; todo evento suscrito, en `eventsConsumed`. Emitir un evento no declarado es un anti-patrón que rechaza la revisión.

## Hooks

Los hooks son **síncronos dentro del flujo** de otro módulo: sirven para que un módulo intervenga en una operación ajena sin que el dueño conozca al invitado.

### Exponer un hook (el dueño)

`mod.courses` declara en su manifest:

```ts
hooksExposed: [
  {
    name: 'courses.publish.validate',
    description: 'Permite añadir validaciones antes de publicar un curso.',
    async: true,
  },
],
```

Y lo dispara en su service:

```ts
await this.ctx.hookRegistry.run('courses.publish.validate', {
  tenantId,
  input: { courseId, reasons: [] },
  metadata: {},
});
// si input.reasons quedó con entradas, la publicación se bloquea
```

### Registrarse a un hook (el invitado)

`mod.fundae` añade su validación sin que `mod.courses` sepa que existe:

```ts
ctx.hookRegistry.register<{ courseId: string; reasons: string[] }>(
  'courses.publish.validate',
  async ({ input }) => {
    if (!hasObjective(input.courseId)) input.reasons.push('Falta objetivo Fundae');
  },
);
```

Los handlers se ejecutan secuencialmente; comunican el resultado mutando el `input` compartido (el patrón `reasons[]` para vetos).

## Eventos y hooks vs. services públicos

Cuando necesitas **datos** de otro módulo de forma síncrona, la vía preferente no es un hook sino su **service público** inyectado por el host (vía A del ADR-016) — o, como último recurso, lectura directa de sus tablas con la dependencia declarada (vía B, solo lectura). Ver [Base de datos](base-de-datos.md#leer-datos-de-otro-modulo-via-b-del-adr-016).

## Webhooks salientes

Los eventos de dominio marcados en el catálogo de webhooks (`learning.enrollment.created`, `community.post.created`, `billing.subscription.*`…) se reenvían además a los **webhooks salientes** que configuren los administradores. Si tu evento debería ser suscribible desde fuera, propón añadirlo al catálogo. Ver [API → Convenciones](../../api/convenciones.md#webhooks-salientes).
