# Backend del módulo

El backend de un módulo tiene dos mitades: el **paquete puro** (`modules/<slug>/`) con la lógica de dominio, y el **host NestJS** (`apps/api/src/modules/<slug>/`) que lo expone por HTTP y le cablea la infraestructura.

## El contrato `DidactaModule`

`src/index.ts` del paquete exporta el módulo con sus **lifecycle hooks**:

```ts
import type { DidactaModule, ModuleContext } from '@didacta/core-kernel';
import { manifest } from './manifest';

export const miModulo: DidactaModule = {
  manifest,

  async onRegister(ctx: ModuleContext) {
    // 1 vez por proceso, sin tenant. Aquí van las suscripciones al event bus.
  },
  async onEnable(tenantId: string, ctx: ModuleContext) {
    // Cada vez que un tenant activa el módulo.
  },
  async onDisable(tenantId: string, ctx: ModuleContext) {
    // El tenant lo desactiva. Los datos SE CONSERVAN.
  },
  async onUninstall(tenantId: string, ctx: ModuleContext) {
    // Desinstalación. Los datos se archivan, no se borran.
  },
};

export { manifest, MiModuloService };
```

Dos patrones de export, ambos en uso:

- **Objeto directo** (`export const coursesModule: DidactaModule = {...}`) — cuando el módulo no necesita nada del host para construirse.
- **Factory** (`export function buildAccessGroupsModule(deps): DidactaModule`) — cuando necesita inyección desde el host (adapters de Stripe, resolvers de usuarios, funciones de IA…).

!!! warning "Las suscripciones de eventos no se apagan solas"
    `onRegister` corre una vez por proceso y sin tenant; `onDisable` **no desuscribe** del bus. Si tu handler reacciona a eventos, comprueba dentro si el módulo está activo para el `tenantId` del evento antes de tocar datos.

## El `ModuleContext`

El módulo nunca crea infraestructura: la recibe.

| Servicio | Para qué |
| --- | --- |
| `eventBus` | Publicar y suscribirse a eventos de dominio (outbox transaccional). |
| `hookRegistry` | Registrar y disparar hooks síncronos entre módulos. |
| `storage` | Subir ficheros e imágenes (con optimización automática). |
| `auditLog` | Registrar acciones auditables. |
| `evidenceVault` | Evidencias con integridad (Fundae, cumplimiento). |
| `notificationHub` | Notificar por email / in-app / webhook respetando las preferencias del usuario. |
| `i18n` | Textos localizados. |
| `logger` | Logging estructurado (Pino). |
| `config` | Configuración por `(tenantId, módulo, clave)`. |

## Registro en el arranque

El registro de módulos es **explícito** (no hay autodiscovery de carpetas): añade tu módulo al array de `registry.register([...])` en `apps/api/src/modules/module-registry.service.ts`. En ese mismo fichero el host construye tus dependencias (services de dominio, puertos de IA, publishers) si usas el patrón factory.

En el arranque, el registro:

1. Valida `coreVersionRequired` contra la versión del core.
2. Ordena los módulos **topológicamente** por dependencias (detecta ciclos y versiones incompatibles con errores tipados).
3. Ejecuta `onRegister` de cada uno.
4. Persiste los manifests en la tabla `module` (upsert por nombre).

## El host NestJS

En `apps/api/src/modules/<slug>/` viven los adaptadores HTTP:

```
apps/api/src/modules/access-groups/
├── access-groups.controller.ts        # rutas bajo /modules/access-groups
├── access-groups.service.ts           # service con Prisma real
├── access-groups-courses.bridge.ts    # suscriptor de courses.course.published
└── access-groups-tiers.bridge.ts      # suscriptor de payment_connections.user_tier.changed
```

Reglas del host:

- El controller usa `@Controller('modules/<slug>')` — coherente con el `apiNamespace` del manifest. El interceptor de acceso por módulo gatea esas rutas automáticamente cuando el tenant lo tiene desactivado.
- **Sin lógica de negocio en el controller**: valida con Zod (`ZodValidationPipe` por parámetro), delega en el service.
- Los errores de dominio se mapean a HTTP con un **exception filter** propio del módulo (patrón `courses-error.filter.ts`), registrado en el módulo NestJS correspondiente.
- Declara el módulo NestJS en `apps/api/src/modules/modules.module.ts`.

## Módulos condicionados por entorno

Si tu módulo depende de configuración externa (como `mod.billing` de `STRIPE_SECRET_KEY`), el patrón es: **no registrarse** si falta la configuración y responder `503` con un error claro en sus endpoints — nunca tumbar el arranque del resto de la plataforma.
