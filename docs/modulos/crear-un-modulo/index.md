# Crear un módulo

Guía completa para desarrollar un módulo first-party de Didacta. La plantilla de referencia es `modules/hello-world` — cópiala y adapta.

## Anatomía de un módulo

Un módulo first-party se reparte en capas con responsabilidades claras:

| Capa | Ruta | Qué contiene |
| --- | --- | --- |
| **Paquete del módulo** | `modules/<slug>/` | Lógica de dominio **pura**: manifest, schemas Zod, servicios sin NestJS ni Prisma. Testeable de forma aislada. |
| **Host NestJS** | `apps/api/src/modules/<slug>/` | Controllers, services con Prisma, bridges de eventos, workers. |
| **Tablas** | `packages/database/prisma/schema.prisma` | Modelos con `@@map("mod_<slug>_…")` + migración versionada. |
| **UI** | `apps/web/src/modules/<slug>/` + páginas en `apps/web/src/app/(app)/` | Extensión web (menú, tabs de configuración) y páginas Next.js. |

## Estructura mínima del paquete

```
modules/mi-modulo/
├── module.json          # manifest para el linter (module-doctor)
├── package.json         # @didacta/mod-mi-modulo
├── README.md            # con las 9 secciones obligatorias
├── tsconfig.json        # extends ../../tsconfig.base.json
├── vitest.config.ts
├── src/
│   ├── manifest.ts      # parseModuleManifest(...) → export const manifest
│   ├── service.ts       # lógica de dominio
│   └── index.ts         # export const miModulo: DidactaModule
└── tests/
    └── contract.test.ts
```

`package.json` del paquete (patrón a copiar):

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

## Checklist completo

1. Crea `modules/<slug>/` con la estructura de arriba (el workspace pnpm ya incluye `modules/*`).
2. Escribe el [manifest](manifest.md) en `src/manifest.ts` con `parseModuleManifest(...)`.
3. Exporta el contrato [`DidactaModule`](backend.md#el-contrato-didactamodule) desde `src/index.ts`.
4. Regístralo en el array de `registry.register([...])` de `apps/api/src/modules/module-registry.service.ts` (el orden no importa: hay orden topológico automático).
5. Crea el [host NestJS](backend.md) en `apps/api/src/modules/<slug>/` y decláralo en `apps/api/src/modules/modules.module.ts`.
6. Añade las [tablas](base-de-datos.md) al schema de Prisma con prefijo `mod_<slug>_` + `tenant_id`, y genera la migración versionada.
7. Crea la [extensión de UI](ui.md) en `apps/web/src/modules/<slug>/index.ts` y regístrala en `apps/web/src/modules/index.ts`.
8. Completa `module.json` y el `README.md` con las 9 secciones, y pasa las [validaciones](validacion.md).

## Las reglas de oro

Del contrato de módulo ([reglas completas](../index.md#reglas-de-aislamiento)):

- Nunca importes código de otro módulo.
- Nunca escribas en tablas de otro módulo (leer sí, declarando la dependencia y filtrando por `tenant_id`).
- Nunca modifiques el core para una feature de tu módulo.
- Nunca emitas eventos sin declararlos en el manifest.
- Nunca crees FKs hacia tablas de otro módulo.
- Nunca metas lógica de negocio en controllers.
- Nunca gatees un módulo por licencia: **todos los módulos son Community**.

## Guía paso a paso

1. [El manifest](manifest.md) — el contrato declarativo del módulo.
2. [Base de datos](base-de-datos.md) — tablas, RLS y migraciones.
3. [Backend](backend.md) — el `DidactaModule`, el `ModuleContext` y el host NestJS.
4. [Interfaz de usuario](ui.md) — menú, páginas y tabs de configuración.
5. [Eventos y hooks](eventos-y-hooks.md) — comunicación con otros módulos.
6. [Validación y tests](validacion.md) — module-doctor, README y suites.
