# Building a module

A complete guide to developing a first-party Didacta module. The reference template is `modules/hello-world` — copy it and adapt.

## Anatomy of a module

A first-party module is split into layers with clear responsibilities:

| Layer | Path | What it contains |
| --- | --- | --- |
| **Module package** | `modules/<slug>/` | **Pure** domain logic: manifest, Zod schemas, services free of NestJS and Prisma. Testable in isolation. |
| **NestJS host** | `apps/api/src/modules/<slug>/` | Controllers, Prisma services, event bridges, workers. |
| **Tables** | `packages/database/prisma/schema.prisma` | Models with `@@map("mod_<slug>_…")` + a versioned migration. |
| **UI** | `apps/web/src/modules/<slug>/` + pages under `apps/web/src/app/(app)/` | The web extension (menu, settings tabs) and the Next.js pages. |

## Minimum package structure

```
modules/my-module/
├── module.json          # manifest for the linter (module-doctor)
├── package.json         # @didacta/mod-my-module
├── README.md            # with the 9 mandatory sections
├── tsconfig.json        # extends ../../tsconfig.base.json
├── vitest.config.ts
├── src/
│   ├── manifest.ts      # parseModuleManifest(...) → export const manifest
│   ├── service.ts       # domain logic
│   └── index.ts         # export const myModule: DidactaModule
└── tests/
    └── contract.test.ts
```

The package's `package.json` (the pattern to copy):

```json
{
  "name": "@didacta/mod-mi-modulo",
  "version": "1.0.0",
  "private": true,
  "main": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "scripts": {
    "build": "tsc -p tsconfig.json",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run"
  },
  "dependencies": { "@didacta/core-kernel": "workspace:*" },
  "devDependencies": { "@didacta/core-registry": "workspace:*", "vitest": "^2.1.8", "typescript": "^5.7.2" }
}
```

## Full checklist

1. Create `modules/<slug>/` with the structure above (the pnpm workspace already includes `modules/*`).
2. Write the [manifest](manifest.md) in `src/manifest.ts` with `parseModuleManifest(...)`.
3. Export the [`DidactaModule`](backend.md#the-didactamodule-contract) contract from `src/index.ts`.
4. Register it in the `registry.register([...])` array in `apps/api/src/modules/module-registry.service.ts` (order does not matter: topological ordering is automatic).
5. Create the [NestJS host](backend.md) in `apps/api/src/modules/<slug>/` and declare it in `apps/api/src/modules/modules.module.ts`.
6. Add the [tables](base-de-datos.md) to the Prisma schema with the `mod_<slug>_` prefix + `tenant_id`, and generate the versioned migration.
7. Create the [UI extension](ui.md) in `apps/web/src/modules/<slug>/index.ts` and register it in `apps/web/src/modules/index.ts`.
8. Complete `module.json` and the `README.md` with the 9 sections, and pass the [validations](validacion.md).

## The golden rules

From the module contract ([full rules](../index.md#isolation-rules)):

- Never import another module's code.
- Never write to another module's tables (reading is fine, declaring the dependency and filtering by `tenant_id`).
- Never modify the core for one of your module's features.
- Never emit events without declaring them in the manifest.
- Never create foreign keys pointing at another module's tables.
- Never put business logic in controllers.
- Never gate a module behind a license: **every module is Community**.

## Step-by-step guide

1. [The manifest](manifest.md) — the module's declarative contract.
2. [Database](base-de-datos.md) — tables, RLS and migrations.
3. [Backend](backend.md) — the `DidactaModule`, the `ModuleContext` and the NestJS host.
4. [User interface](ui.md) — menu, pages and settings tabs.
5. [Events and hooks](eventos-y-hooks.md) — communicating with other modules.
6. [Validation and tests](validacion.md) — module-doctor, the README and the test suites.
