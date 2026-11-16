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
    userId?: string;     // el ACTOR, que no siempre es el sujeto del evento
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

Convención de nombres: `<ámbito>.<entidad>.<acción>` (`courses.course.published`, `subscriptions.subscription.canceled`, `learning.course.completed`). El ámbito suele ser el slug del módulo, pero no siempre: `mod.zoom-live` emite `zoom.session.*`, `mod.member-registration` emite `member_registration.*` y `mod.payment-connections` emite `payment_connections.*`.

### Entrega garantizada: transactional outbox

`publish()` no lanza el evento directamente: lo persiste en la tabla `outbox_event` (deduplicando por `(tenantId, idempotencyKey)`) y un worker lo despacha vía BullMQ. Sin Redis, el despacho es in-process inmediato; un worker de recuperación reencola los pendientes. Resultado: un evento publicado dentro de una transacción **no se pierde** aunque el proceso caiga.

Cada fila de `outbox_event` está en uno de cuatro estados:

| `processed_at` | `last_error` | Significado |
|---|---|---|
| con valor | vacío | Entregado a al menos un handler, todos correctos |
| con valor | `NO_HANDLER: <evento>` | **No llegó a nadie.** Es un estado terminal etiquetado, no un éxito: si en las siguientes 24 horas aparece un suscriptor, se reentrega |
| vacío | vacío | Pendiente: encolado, aún sin despachar |
| vacío | un error | Falló algún handler; se reintenta |

Un evento sin suscriptores es legítimo (un bus puede tener fan-out a cero), pero conviene saber que **«entregado» y «sin destinatario» no son lo mismo**.

### La clave de idempotencia

Calcula la `idempotencyKey` con el helper del kernel, no a mano:

```ts
import { eventIdempotencyKey } from '@didacta/core-kernel';

idempotencyKey: eventIdempotencyKey('community.post.created', data /*, uniqueSuffix? */),
```

Devuelve `<nombre>:<resumen del payload>:<sufijo>` y **nunca pasa de 256 caracteres**. Componerla a mano con el payload entero dentro (`${name}:${JSON.stringify(data)}:${Date.now()}`) revienta el índice único de la tabla cuando el payload crece: la acción de negocio ya está hecha y el evento —con los avisos, webhooks y matrículas que cuelgan de él— no existe.

- El `uniqueSuffix` por defecto es `Date.now()`. Si el evento tiene un **identificador natural de la ocurrencia** (el pedido, el intento), pásalo como sufijo: la clave es entonces estable y un reintento no publica dos veces.

!!! warning "Elige bien el sufijo"
    La deduplicación por `idempotencyKey` es **para siempre**. Si una entidad puede reemitir legítimamente el mismo estado (una suscripción que se activa, se cancela y se reactiva meses después), el sufijo tiene que ser **por ocurrencia** (`randomUUID()`, o un contador), no estable — o el segundo evento se tragará en silencio.

### Suscribirse

Las suscripciones se hacen en `onRegister` (una vez por proceso, sin tenant) o en un *bridge* del host (`apps/api/src/modules/<slug>/*.bridge.ts`, que es donde un módulo reacciona a otro sin importarlo):

```ts
// apps/api/src/modules/access-groups/access-groups-courses.bridge.ts (patrón)
eventBus.subscribe('courses.course.published', async (event) => {
  // 1) ¿Está mi módulo activo para event.metadata.tenantId? Si no, return.
  // 2) Reaccionar (filtrando SIEMPRE por tenantId).
});
```

Recuerda: desactivar un módulo **no** desuscribe sus handlers — comprueba el estado dentro del handler. Los handlers de un mismo evento se ejecutan todos aunque uno falle, y siempre dentro del contexto del tenant del evento. Si alguno lanza, el despacho entero se marca como fallido y se reintenta —**también los handlers que ya habían terminado bien**—, así que **todo handler debe ser idempotente**.

### Declaración en el manifest

Todo evento publicado va en `eventsEmitted`; todo evento suscrito, en `eventsConsumed`. Emitir un evento no declarado es un anti-patrón que rechaza la revisión.

!!! note "Módulos de terceros (sandbox)"
    El manifest de un módulo del marketplace también lleva `eventsEmitted` y `eventsConsumed`, pero el host todavía no expone un cliente de eventos (`ctx.events`) a los módulos que corren en el sandbox: de momento declarar eventos ahí no tiene efecto. Los módulos de terceros se integran por las [APIs públicas del core](../third-party-modules.md).

## Catálogo de eventos de dominio

Es lo que declara cada módulo in-tree en su manifest, y coincide con lo que el código publica: `scripts/module-doctor.ts` (CI) falla si un módulo declara un evento que no aparece en ningún `publish()`, o si el código de un módulo menciona un evento propio que no está declarado. Es una comprobación estática: confirma que el evento existe en el código, no que cada ruta lo ejecute.

| Módulo | Emite | Consume |
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

**Lo que no hay a propósito.** `mod.wp-sso` no emite eventos al bus: el intercambio de token deja su rastro en el **registro de auditoría** (`sso.wp.signin.success`, `sso.wp.user.provisioned`, `sso.wp.account.linked`, `sso.wp.callback.rejected`). Tampoco existe un `subscriptions.membership.revoked`: la pérdida de acceso de una membresía viaja en `subscriptions.subscription.canceled` (cancelación inmediata), `.unpaid` y `.paused`, con `planId` en el payload. Y un turno del tutor es **un** evento, `ai-tutor.chat.message-received`; no hay un `message-sent` aparte.

**Payloads nuevos.** `certificates.revoked` lleva `{ userId, courseId, reason, revokedCount }` y se publica al revocar los certificados de un curso (hoy, al reembolsarse la compra); `ai-grader.suggestion.generated` lleva `{ suggestionId, attemptId, answerId, questionId, proposedScore, provider, model }` y `ai-grader.suggestion.failed` lleva `{ attemptId, answerId, questionId, reason }`, con `reason` = `provider` o `unparseable`. `learning.enrollment.paused` y `.resumed` se publican solo si alguna matrícula cambió de estado de verdad. Al añadir un evento, declara **siempre** en el manifest lo que publicas.

El nombre y la forma de cada evento los define el módulo que lo emite; mira su `data` en el servicio que llama a `publish()`.

## Hooks

Los hooks son **síncronos dentro del flujo** de otro módulo: sirven para que un módulo intervenga en una operación ajena sin que el dueño conozca al invitado.

```ts
interface HookRegistry {
  register<TInput>(hookName: string, handler: (ctx: HookContext<TInput>) => Promise<void> | void): void;
  run<TInput>(hookName: string, ctx: HookContext<TInput>): Promise<void>;
}

interface HookContext<TInput> {
  tenantId: string;
  input: TInput;                    // el objeto que el dueño comparte y los handlers pueden mutar
  metadata: Record<string, unknown>;
}
```

### Los hooks que existen hoy

| Hook | Lo expone | Cuándo corre | `input` |
|---|---|---|---|
| `courses.publish.validate` | `mod.courses` | Al publicar un curso, justo antes de cambiar su estado (y después de comprobar que tiene lecciones) | `{ courseId, reasons: string[] }`; `metadata.actorId` |

Es el **único** hook que expone un módulo, y en esta versión **ningún módulo registra un handler** para él: es un punto de extensión abierto, no un mecanismo en uso.

### Exponer un hook (el dueño)

`mod.courses` lo declara en su manifest:

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
const reasons: string[] = [];
await this.ctx.hookRegistry.run('courses.publish.validate', {
  tenantId,
  input: { courseId, reasons },
  metadata: { actorId },
});
if (reasons.length > 0) throw new PublishValidationError(reasons);
// si reasons quedó con entradas, la publicación se bloquea
```

### Registrarse a un hook (el invitado)

Un módulo registra su handler en `onRegister`, sin que el dueño sepa que existe (patrón; hoy nadie lo usa para este hook):

```ts
ctx.hookRegistry.register<{ courseId: string; reasons: string[] }>(
  'courses.publish.validate',
  async ({ input }) => {
    if (!cumpleMiRequisito(input.courseId)) input.reasons.push('Falta mi requisito');
  },
);
```

Reglas del registro:

- Los handlers se ejecutan **secuencialmente**, en orden de registro, y comunican el resultado **mutando el `input` compartido** (el patrón `reasons[]` para vetos).
- Un handler que **lanza** interrumpe la operación del dueño con ese error: para vetar, añade un motivo a `reasons`, no lances.
- El registro es **en memoria y por proceso**, sin distinción de tenant: igual que en los eventos, comprueba dentro del handler que tu módulo esté activo para `ctx.tenantId`.
- Un hook sin handlers no hace nada.

## Eventos y hooks vs. services públicos

Cuando necesitas **datos** de otro módulo de forma síncrona, la vía preferente no es un hook sino su **service público** inyectado por el host (vía A del ADR-016) — o, como último recurso, lectura directa de sus tablas con la dependencia declarada (vía B, solo lectura). Ver [Base de datos](database.md#leer-datos-de-otro-modulo-via-b-del-adr-016).

## Webhooks salientes

Un evento de dominio **no sale fuera** automáticamente. Solo se reenvían a los webhooks salientes que configuran los administradores los eventos que figuran en el **catálogo de webhooks** del host (`KNOWN_EVENT_TYPES` en `apps/api/src/webhooks/webhooks.types.ts`): `learning.enrollment.*`, `learning.course.completed`, `community.post.created`, `community.comment.created`, los pedidos (`billing.order.*`, no todos), el ciclo de suscripción y membresía (`subscriptions.*`, no todos), `fundae.group.closed` y `assessments.attempt.submitted`. La lista completa, con su payload, está en [API → Webhooks](../../api/webhooks.md#catalogo-de-eventos).

Si tu evento debería ser suscribible desde fuera, propón añadirlo al catálogo. Esa lista es **doblemente vinculante** —el puente solo se suscribe a lo que hay en ella y la interfaz solo ofrece eso—, así que antes de añadir una entrada comprueba que hay un `publish()` real detrás: el manifest declara lo que el módulo se compromete a emitir y `module-doctor` vigila que el evento exista en el código, pero no prueba que cada ruta lo publique (hubo un `subscriptions.membership.revoked` declarado que nadie publicaba, y se retiró). Verifica el `publish()`.
