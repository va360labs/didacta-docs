# The manifest

The manifest is the module's **declarative contract**: identity, version, tables, permissions, dependencies and events. It is defined in `src/manifest.ts` and validated with the Zod schema from `@didacta/core-kernel` **at import time** — an invalid manifest stops the API from starting, so mistakes surface immediately.

## Minimal example (hello-world)

```ts
import { parseModuleManifest, type ModuleManifest } from '@didacta/core-kernel';

export const manifest: ModuleManifest = parseModuleManifest({
  name: 'mod.hello-world',
  displayName: 'Hello World',
  description: 'Módulo de ejemplo. Plantilla de referencia para nuevos módulos.',
  version: '1.0.0',
  author: 'VA360 LABS',
  license: 'Proprietary',
  category: 'example',
  coreVersionRequired: '^1.0.0',
  tablePrefix: 'mod_helloworld_',
  permissions: ['hello-world.greeting.read'],
  eventsEmitted: ['hello-world.greeting.requested'],
  eventsConsumed: [],
  apiNamespace: '/modules/hello-world',
});
```

## Fields

| Field | Required | Format | What it is |
| --- | --- | --- | --- |
| `name` | ✔ | `mod.<slug>` (`^mod\.[a-z0-9-]+$`) | The module's unique identifier. |
| `displayName` | ✔ | text | The name shown in the panel. |
| `description` | ✔ | text | What the module does, in one sentence. |
| `version` | ✔ | SemVer `X.Y.Z` | The module version. |
| `coreVersionRequired` | ✔ | SemVer range (`^1.0.0`) | The core version it needs; validated on registration. |
| `tablePrefix` | ✔ | `mod_<slug>_` (`^mod_[a-z0-9_]+_$`) | The prefix of **all** its tables. |
| `apiNamespace` | ✔ | starts with `/` (convention: `/modules/<slug>`) | The prefix of its endpoints; used by per-tenant gating. |
| `category` | — | text | `'core'` marks the module as **not disableable**. Others in use: `ai`, `engagement`, `compliance`, `live`, `integration`, `migration`. |
| `dependencies.modules` | — | `{ name, version }[]` | **Hard** dependencies: without them the module does not start (registration error). |
| `dependencies.optionalModules` | — | `{ name, version }[]` | **Soft** dependencies: they document optional integrations. |
| `permissions` | — | `string[]` | Permissions the module defines (`<slug>.<resource>.<action>`). |
| `eventsEmitted` | — | `string[]` | Domain events it publishes. **Every published event must be listed here.** |
| `eventsConsumed` | — | `string[]` | Other modules' events it subscribes to. |
| `hooksExposed` | — | `{ name, description?, async }[]` | Extension points it offers to other modules. |
| `author`, `license` | — | text | Metadata. |

!!! note "Declarative fields with no runtime effect (today)"
    The schema also accepts `roles`, `defaultConfig`, `uiExtensions`, `pages` and `hooksConsumed`, but no component consumes them right now: in-tree UI is declared through the [web registry](ui.md) and system roles are fixed. You can declare them as documentation, knowing they have no effect yet.

## Full example with dependencies and events (access-groups)

```ts
export const manifest: ModuleManifest = parseModuleManifest({
  name: 'mod.access-groups',
  displayName: 'Grupos de acceso',
  description: 'Grupos configurables que otorgan acceso a un set de cursos…',
  version: '1.0.0',
  author: 'VA360 LABS',
  license: 'Proprietary',
  category: 'core',
  coreVersionRequired: '^1.0.0',
  tablePrefix: 'mod_access_groups_',
  permissions: [
    'access_groups.group.read',
    'access_groups.group.manage',
    'access_groups.member.manage',
  ],
  dependencies: {
    modules: [                                    // hard
      { name: 'mod.courses',  version: '^1.0.0' },
      { name: 'mod.learning', version: '^1.0.0' },
    ],
    optionalModules: [                            // soft
      { name: 'mod.payment-connections', version: '^1.0.0' },
      { name: 'mod.subscriptions',       version: '^1.0.0' },
    ],
  },
  eventsEmitted: [],
  eventsConsumed: [
    'courses.course.published',
    'payment_connections.user_tier.changed',
    'subscriptions.membership.activated',
    'subscriptions.subscription.activated',
    'subscriptions.subscription.canceled',
    'subscriptions.subscription.unpaid',
  ],
  apiNamespace: '/modules/access-groups',
});
```

## Declaring a hook

The only module that exposes a hook today is `mod.courses` — it serves as the pattern:

```ts
hooksExposed: [
  {
    name: 'courses.publish.validate',
    description: 'Permite que otros módulos añadan validaciones antes de publicar un curso (ej. mod.fundae verifica objetivos y duración).',
    async: true,
  },
],
```

How it is consumed and fired is covered in [Events and hooks](eventos-y-hooks.md).

## The second manifest: `module.json`

Every module also carries a `module.json` at its root, used by the `scripts/module-doctor.ts` linter (not by the runtime). The fields it requires: `name`, `version`, `edition` (always `community`), `coreVersionRequired`, `tablePrefix`, `apiNamespace`. **Marketplace-style** `module.json` files (with `vendor`/`isolation`/`http`/`didacta`, the expanded shape validated by the host's strict schema) are the exception: they carry no `edition` — the host schema rejects that key on install. Keep it **consistent with `src/manifest.ts`** — the doctor detects divergences. See [Validation](validacion.md).
