# mod.hello-world — Módulo de referencia

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **example** (desactivable)

## Qué hace

Módulo de ejemplo y **plantilla de partida** para crear módulos nuevos. Demuestra los cuatro elementos del contrato:

1. Un **manifest** válido parseado con `parseModuleManifest` del core-kernel.
2. Los cuatro **lifecycle hooks** (`onRegister`, `onEnable`, `onDisable`, `onUninstall`), con comentarios de qué haría un módulo real en cada uno.
3. Un **service de dominio** (`HelloWorldService`) que recibe el `ModuleContext` y consume `eventBus` e `i18n`.
4. Una **suite de tests de contrato** que registra el módulo en un `ModuleRegistry` real y lo verifica end-to-end.

## Cómo usarlo

El procedimiento para arrancar un módulo nuevo: copiar la carpeta, renombrar `name`/`tablePrefix`/`apiNamespace`/permisos, escribir el dominio, implementar el lifecycle y actualizar los tests — la guía completa está en [Crear un módulo](../crear-un-modulo/index.md).

Su README enumera los cuatro anti-patrones que todo módulo respeta: no importar código de otro módulo, no tocar tablas ajenas con Prisma, no modificar el core para features propias, y no emitir eventos sin declararlos.

## Datos, API y eventos

- Declara `tablePrefix` pero **no crea tablas** (no las necesita).
- Declara `apiNamespace /modules/hello-world` pero no expone endpoints reales.
- Declara el evento `hello-world.greeting.requested` y el permiso `hello-world.greeting.read` como ejemplo del contrato.

## Configuración

Ninguna.
