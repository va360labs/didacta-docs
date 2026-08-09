# The module's backend

A module's backend has two halves: the **pure package** (`modules/<slug>/`) holding the domain logic, and the **NestJS host** (`apps/api/src/modules/<slug>/`) that exposes it over HTTP and wires up its infrastructure.

## The `DidactaModule` contract

The package's `src/index.ts` exports the module with its **lifecycle hooks**:

```ts
import type { DidactaModule, ModuleContext } from '@didacta/core-kernel';
import { manifest } from './manifest';

export const miModulo: DidactaModule = {
  manifest,

  async onRegister(ctx: ModuleContext) {
    // Once per process, with no tenant. Event bus subscriptions go here.
  },
  async onEnable(tenantId: string, ctx: ModuleContext) {
    // Every time a tenant enables the module.
  },
  async onDisable(tenantId: string, ctx: ModuleContext) {
    // The tenant disables it. Data IS PRESERVED.
  },
  async onUninstall(tenantId: string, ctx: ModuleContext) {
    // Uninstall. Data is archived, not deleted.
  },
};

export { manifest, MiModuloService };
```

Two export patterns, both in use:

- **A plain object** (`export const coursesModule: DidactaModule = {...}`) — when the module needs nothing from the host to be constructed.
- **A factory** (`export function buildAccessGroupsModule(deps): DidactaModule`) — when it needs injection from the host (Stripe adapters, user resolvers, AI functions…).

!!! warning "Event subscriptions do not switch themselves off"
    `onRegister` runs once per process and with no tenant; `onDisable` **does not unsubscribe** from the bus. If your handler reacts to events, check inside it whether the module is active for the event's `tenantId` before touching data.

## The `ModuleContext`

A module never creates infrastructure: it receives it.

| Service | What it is for |
| --- | --- |
| `eventBus` | Publishing and subscribing to domain events (transactional outbox). |
| `hookRegistry` | Registering and firing synchronous hooks between modules. |
| `storage` | Uploading files and images (with automatic optimisation). |
| `auditLog` | Recording auditable actions. |
| `evidenceVault` | Evidence with integrity guarantees (Fundae, compliance). |
| `notificationHub` | Notifying by email / in-app / webhook, honouring the user's preferences. |
| `i18n` | Localised copy. |
| `logger` | Structured logging (Pino). |
| `config` | Configuration per `(tenantId, module, key)`. |

## Registration at startup

Module registration is **explicit** (there is no folder autodiscovery): add your module to the `registry.register([...])` array in `apps/api/src/modules/module-registry.service.ts`. In that same file the host builds your dependencies (domain services, AI ports, publishers) if you use the factory pattern.

At startup, the registry:

1. Validates `coreVersionRequired` against the core version.
2. Orders the modules **topologically** by dependency (detecting cycles and incompatible versions with typed errors).
3. Runs each module's `onRegister`.
4. Persists the manifests in the `module` table (upsert by name).

## The NestJS host

`apps/api/src/modules/<slug>/` is where the HTTP adapters live:

```
apps/api/src/modules/access-groups/
├── access-groups.controller.ts        # routes under /modules/access-groups
├── access-groups.service.ts           # service with real Prisma
├── access-groups-courses.bridge.ts    # subscriber to courses.course.published
└── access-groups-tiers.bridge.ts      # subscriber to payment_connections.user_tier.changed
```

Host rules:

- The controller uses `@Controller('modules/<slug>')` — consistent with the manifest's `apiNamespace`. The per-module access interceptor gates those routes automatically when the tenant has the module disabled.
- **No business logic in the controller**: validate with Zod (`ZodValidationPipe` per parameter) and delegate to the service.
- Domain errors are mapped to HTTP with the module's own **exception filter** (the `courses-error.filter.ts` pattern), registered in the corresponding NestJS module.
- Declare the NestJS module in `apps/api/src/modules/modules.module.ts`.

## Modules that depend on external configuration

If your module depends on an external credential (as `mod.billing`/`mod.subscriptions` depend on their Stripe keys), the pattern is: the module **always registers itself** — never gate the whole startup on a missing configuration, because in a multi-tenant setup that configuration may live PER TENANT (the admin panel, an encrypted `tenant_setting`) and you cannot know until the first real call whether a given tenant has it. Instead, resolve the credential **inside each method** (a `stripeFor(tenantId)` or equivalent, never cached at instance level) and throw a typed domain error if it is missing — the module's own exception filter maps it to a `503` with a clear message. That way an unconfigured tenant brings nothing down for the rest, and the same deployment can have tenants with and without the integration enabled at the same time.
