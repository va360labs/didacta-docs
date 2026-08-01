# Interfaz de usuario del módulo

La UI de un módulo first-party vive **in-tree** en la web de Didacta: una extensión declarativa (menú y tabs de configuración) más las páginas Next.js que necesite.

## La extensión web

Cada módulo con UI declara un `ModuleWebExtension` en `apps/web/src/modules/<slug>/index.ts`:

```ts
import type { ModuleWebExtension } from '@/lib/module-registry';

export const miModuloExtension: ModuleWebExtension = {
  name: 'mod.mi-modulo',            // slug EXACTO del manifest
  sidebarItems: [
    {
      group: 'Aprendizaje',
      href: '/admin/mi-modulo',
      label: 'Mi módulo',
      icon: 'Sparkles',
      requiresRole: 'tenant_admin',
    },
  ],
  adminConfigTabs: [
    // tabs que aparecen en /admin/configuracion
  ],
};
```

- `name` debe coincidir **exactamente** con el `name` del manifest — es la clave con la que el frontend filtra por módulos activos.
- `group` coloca el item en una de las secciones del menú lateral: `Aprendizaje`, `Personas`, `Formador`, `Personas y accesos`, `Comunidad`, `Contenido`, `Ingresos`, `Comunicación`, `Marca y ajustes`, `Integraciones y API`, `Seguridad`, `Plataforma`.
- `requiresRole` restringe la entrada de menú (`super_admin`, `tenant_admin`, `formador`).

## Registro

Añade la extensión al catálogo estático `apps/web/src/modules/index.ts`. El filtrado es automático: el frontend consulta `GET /api/v1/me/modules` y solo muestra los items cuyos módulos están activos para el tenant del usuario — nunca renders muertos ni menús rotos.

## Páginas

Las páginas viven en el App Router: `apps/web/src/app/(app)/…` (por ejemplo `admin/grupos-acceso`, `admin/gamificacion`). Convenciones:

- **Todo dato en pantalla viene de la API** (regla dura del proyecto: prohibidos los arrays hardcodeados de contenido, contadores fijos o nombres inventados).
- Stack de UI: React 19 + Tailwind 4 + shadcn/ui; sigue los componentes y patrones existentes.
- El copy es whitelabel: nada específico de una instalación concreta; los textos de marca salen del branding del tenant.

## UI de módulos third-party

Los módulos de marketplace no tocan el árbol de la web: empaquetan bundles JS por superficie (`dist/ui/admin.js`…) dentro del ZIP, que el frontend carga dinámicamente. Ver [Módulos de terceros](../modulos-de-terceros.md).
