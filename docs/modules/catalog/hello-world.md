# mod.hello-world — Módulo de referencia

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **example** (desactivable)

## Qué hace

Módulo de ejemplo y **plantilla de partida** para crear módulos nuevos. Demuestra los elementos del contrato:

1. Un **manifest** válido parseado con `parseModuleManifest` del core-kernel.
2. Los cuatro **lifecycle hooks** (`onRegister`, `onEnable`, `onDisable`, `onUninstall`), con comentarios de qué haría un módulo real en cada uno.
3. Un **service de dominio** (`HelloWorldService`) que recibe el `ModuleContext` y consume `eventBus` e `i18n`.
4. Una **suite de tests de contrato** que registra el módulo en un `ModuleRegistry` real y lo verifica end-to-end.
5. Una **superficie pública** (`publico`): una página que el módulo publica en la ruta `/hello-world`, renderizada en el servidor y visible sin sesión (ver más abajo).

## Cómo usarlo

El procedimiento para arrancar un módulo nuevo: copiar la carpeta, renombrar `name`/`tablePrefix`/`apiNamespace`/permisos, escribir el dominio, implementar el lifecycle y actualizar los tests — la guía completa está en [Crear un módulo](../building-a-module/index.md).

Las reglas de aislamiento que todo módulo respeta —no importar código de otro módulo, no tocar tablas ajenas con Prisma, no modificar el core para features propias y no emitir eventos sin declararlos— están recogidas en [Crear un módulo → Las reglas de oro](../building-a-module/index.md#las-reglas-de-oro).

## Configuración

Ninguna propia: ni variables de entorno, ni ajustes en el panel, ni capabilities Enterprise. Como módulo **no-core** (categoría `example`), viene registrado en la instancia y aparece con su interruptor en Administración → Marca y ajustes → Configuración, pestaña «Módulos» (`/admin/settings?tab=modules`), donde un admin puede activarlo o desactivarlo por tenant. Su otra superficie es la página pública `/hello-world`, que solo existe en los dominios del aula que sirven el sitio público (ver «Superficie pública»).

!!! note "Capturas en modo completo"
    Las capturas de esta página son del **modo completo** (menú lateral), el de una instalación nueva. En **modo sencillo** las mismas rutas siguen disponibles desde «Más», salvo las pantallas que ese modo sustituye por una versión simplificada (el editor de cursos, Ajustes, Ventas, Alumnos, Directos, la comunidad y Referidos); para usarlas tal como se ven aquí, cambia al modo completo en **Ajustes → «Usar el modo completo»**.

## Uso paso a paso

Es una plantilla para desarrolladores; el flujo es de código, no de panel:

1. Copia la carpeta `modules/hello-world/` como `modules/<tu-slug>/`.
2. En `src/manifest.ts`, renombra `name`, `displayName`, `tablePrefix`, `apiNamespace` y los permisos.
3. Sustituye `HelloWorldService` por tus services de dominio e implementa los cuatro lifecycle hooks de `src/index.ts` con trabajo real (migraciones en `onEnable`, limpieza en `onDisable`…).
4. Actualiza `module.json` y adapta la suite `tests/contract.test.ts` a tu módulo.
5. Regístralo en el `registry.register([...])` de `apps/api/src/modules/module-registry.service.ts` y sigue el resto de la guía [Crear un módulo](../building-a-module/index.md).
6. Para verlo en marcha, entra en `/admin/settings?tab=modules`: «Hello World» aparece listado con su interruptor, igual que aparecerá tu módulo.

![La tarjeta del módulo Hello World con su interruptor en la pestaña Módulos](../../assets/modules/hello-world/hello-world-1-modules.png)

## Superficie pública

`module.json` declara `"surfaces": ["publico"]` y la extensión web del módulo (`apps/web/src/modules/hello-world/`) registra la ruta `/hello-world` con `publicRoutes`. Es el consumidor mínimo de la capacidad «un módulo publica rutas públicas renderizadas en servidor»:

- La página es un componente de **servidor**: su contenido llega en el HTML, sin ejecutar JavaScript, sin sesión y sin llamar a ningún endpoint autenticado. Muestra el nombre del sitio, el dominio, el origen canónico y la ruta.
- La ruta solo se monta si el módulo está **activo para el tenant** y el **dominio de entrada sirve el sitio público**: cada dominio del tenant (`tenant_domain`) tiene una superficie `APP` (el aula con sesión, por defecto) o `SITE` (sitio público). El panel no tiene pantalla para cambiarla; es una columna de esa tabla. Una instalación que no active el módulo no publica nada.
- El origen canónico sale del dominio de entrada, no de `WEB_PUBLIC_URL`; el contexto del sitio (`GET /public/site-context`) responde 404 si el dominio no existe o no sirve el sitio.
- Dos módulos que reclamen la misma ruta pública **rompen la compilación** en vez de resolverse por orden de import, y el esquema del marketplace **rechaza** la superficie `publico` en módulos instalados por ZIP (solo la tienen los módulos incluidos en el build).

## Datos, API y eventos

- Declara `tablePrefix` pero **no crea tablas** (no las necesita).
- Declara `apiNamespace /modules/hello-world` pero no expone endpoints reales.
- Declara el evento `hello-world.greeting.requested` y el permiso `hello-world.greeting.read` como ejemplo del contrato.
