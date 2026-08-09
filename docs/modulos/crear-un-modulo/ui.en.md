# The module's user interface

A first-party module's UI lives **in-tree** in the Didacta web app: a declarative extension (menu and settings tabs) plus whatever Next.js pages it needs.

## The web extension

Every module with a UI declares a `ModuleWebExtension` in `apps/web/src/modules/<slug>/index.ts`:

```ts
import type { ModuleWebExtension } from '@/lib/module-registry';

export const miModuloExtension: ModuleWebExtension = {
  name: 'mod.mi-modulo',            // the EXACT slug from the manifest
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
    // tabs shown under /admin/configuracion
  ],
};
```

- `name` must match the manifest's `name` **exactly** — it is the key the frontend uses to filter by active modules.
- `group` places the item in one of the sidebar sections. The values are internal identifiers and are written in Spanish in the code: `Aprendizaje` (Learning), `Personas` (People), `Formador` (Instructor), `Personas y accesos` (People & access), `Comunidad` (Community), `Contenido` (Content), `Ingresos` (Revenue), `Comunicación` (Communication), `Marca y ajustes` (Brand & settings), `Integraciones y API` (Integrations & API), `Seguridad` (Security), `Plataforma` (Platform). The label the user sees is resolved from the translation catalog.
- `requiresRole` restricts the menu entry (`super_admin`, `tenant_admin`, `formador`).

## Registration

Add the extension to the static catalog in `apps/web/src/modules/index.ts`. Filtering is automatic: the frontend calls `GET /api/v1/me/modules` and only shows the items whose modules are active for the user's tenant — never a dead render, never a broken menu.

## Pages

Pages live in the App Router: `apps/web/src/app/(app)/…` (for example `admin/grupos-acceso`, `admin/gamificacion`). Conventions:

- **Every piece of data on screen comes from the API** (a hard project rule: no hardcoded content arrays, no fixed counters, no made-up names).
- UI stack: React 19 + Tailwind 4 + shadcn/ui; follow the existing components and patterns.
- Copy is whitelabel: nothing specific to a particular installation; brand text comes from the tenant's branding.

## Third-party module UI

Marketplace modules never touch the web tree: they package per-surface JS bundles (`dist/ui/admin.js`…) inside the ZIP, which the frontend loads dynamically. See [Third-party modules](../modulos-de-terceros.md).
