# El manifest

El manifest es el **contrato declarativo** del módulo: identidad, versión, tablas, permisos, dependencias y eventos. Se define en `src/manifest.ts` y se valida con el schema Zod de `@didacta/core-kernel` **en tiempo de import** — un manifest inválido impide arrancar la API, así que los errores se ven al instante.

## Ejemplo mínimo (hello-world)

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

## Campos

| Campo | Obligatorio | Formato | Qué es |
| --- | --- | --- | --- |
| `name` | ✔ | `mod.<slug>` (`^mod\.[a-z0-9-]+$`) | Identificador único del módulo. |
| `displayName` | ✔ | texto | Nombre visible en el panel. |
| `description` | ✔ | texto | Qué hace el módulo, en una frase. |
| `version` | ✔ | SemVer `X.Y.Z` | Versión del módulo. |
| `coreVersionRequired` | ✔ | rango SemVer (`^1.0.0`) | Versión del core que necesita; se valida al registrar. |
| `tablePrefix` | ✔ | `mod_<slug>_` (`^mod_[a-z0-9_]+_$`) | Prefijo de **todas** sus tablas. |
| `apiNamespace` | ✔ | empieza por `/` (convención: `/modules/<slug>`) | Prefijo de sus endpoints; lo usa el gating por tenant. |
| `category` | — | texto | `'core'` marca el módulo como **no desactivable**. Otras usadas: `ai`, `engagement`, `compliance`, `live`, `integration`, `migration`. |
| `dependencies.modules` | — | `{ name, version }[]` | Dependencias **duras**: sin ellas el módulo no arranca (error de registro). |
| `dependencies.optionalModules` | — | `{ name, version }[]` | Dependencias **blandas**: documentan integraciones opcionales. |
| `permissions` | — | `string[]` | Permisos que define el módulo (`<slug>.<recurso>.<acción>`). |
| `eventsEmitted` | — | `string[]` | Eventos de dominio que publica. **Todo evento publicado debe estar aquí.** |
| `eventsConsumed` | — | `string[]` | Eventos de otros módulos a los que se suscribe. |
| `hooksExposed` | — | `{ name, description?, async }[]` | Puntos de extensión que ofrece a otros módulos. |
| `author`, `license` | — | texto | Metadatos. |

!!! note "Campos declarativos sin efecto en runtime (hoy)"
    El schema admite también `roles`, `defaultConfig`, `uiExtensions`, `pages` y `hooksConsumed`, pero actualmente ningún componente los consume: la UI in-tree se declara con el [registro web](ui.md) y los roles del sistema son fijos. Puedes declararlos como documentación, sabiendo que aún no tienen efecto.

## Ejemplo completo con dependencias y eventos (access-groups)

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
    modules: [                                    // duras
      { name: 'mod.courses',  version: '^1.0.0' },
      { name: 'mod.learning', version: '^1.0.0' },
    ],
    optionalModules: [                            // blandas
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

## Declarar un hook

El único módulo que expone un hook hoy es `mod.courses` — sirve de patrón:

```ts
hooksExposed: [
  {
    name: 'courses.publish.validate',
    description: 'Permite que otros módulos añadan validaciones antes de publicar un curso (ej. mod.fundae verifica objetivos y duración).',
    async: true,
  },
],
```

Cómo se consume y se dispara, en [Eventos y hooks](eventos-y-hooks.md).

## El segundo manifest: `module.json`

Cada módulo lleva además un `module.json` en su raíz, usado por el linter `scripts/module-doctor.ts` (no por el runtime). Campos que exige: `name`, `version`, `edition` (siempre `community`), `coreVersionRequired`, `tablePrefix`, `apiNamespace`. Mantenlo **coherente con `src/manifest.ts`** — el doctor detecta divergencias. Ver [Validación](validacion.md).
