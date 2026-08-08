# How modules work

All of Didacta's functionality beyond the core — courses, assessments, certificates, community, payments, virtual classroom… — lives in **modules**. The core provides the platform (tenancy, auth, storage, events, auditing); the modules provide the product.

!!! info "Modules are always Community"
    No module is ever locked behind a license. The only paid features in Didacta are [cross-cutting core capabilities](../enterprise/capabilities.md). This is a non-negotiable design rule of the project.

## Two levels of module

Didacta distinguishes two levels with different contracts:

| | **First-party (in-tree)** | **Third-party (marketplace)** |
| --- | --- | --- |
| Where it lives | In the repository, under `modules/<slug>/` | In a **signed ZIP** uploaded by a `super_admin` |
| Trust | Product code, reviewed in the repository | External code, executed in a **sandbox** (isolated VM) |
| Data access | The host's Prisma; it may **read** (never write) other modules' tables by declaring the dependency | Only its own tables, through a sandboxed `db` with a prefix-based SQL guard |
| UI | In-tree, in the Didacta web app | JS bundles inside the ZIP |

The 24 modules Didacta ships are first-party. The third-party level exists for the future marketplace — see [Third-party modules](modulos-de-terceros.md).

## Isolation rules

Every module, whatever its level, honours the same contract:

- **Its own tables**, prefixed `mod_<slug>_`, with a `tenant_id` column + an RLS policy (automatic — see [Database](../configuracion/base-de-datos.md)).
- **Zero foreign keys** between tables of different modules.
- **Zero imports** of another module's private code.
- Communication between modules goes through **domain events**, **hooks** or **public APIs** — and every event emitted or consumed is declared in the manifest.
- A first-party module may **read** another module's tables if it declares the dependency in its manifest and always filters by `tenant_id`. Writing, never.

## The module context

A module does not reach infrastructure directly: it receives a `ModuleContext` carrying the core services —

`eventBus` (domain events with a transactional outbox) · `hookRegistry` (synchronous extension points) · `storage` · `auditLog` · `evidenceVault` · `notificationHub` (email / in-app / webhook, honouring preferences) · `i18n` · `logger` · `config` (per-tenant, per-module configuration).

## Activation per tenant

Modules are enabled and disabled **per organization**:

- The state lives in the database (`tenant_module`); with no record, the module's `enabledByDefault` applies.
- Modules in the **core** category (courses, learning, assessments, certificates…) cannot be disabled (the API returns **422** `CORE_MODULE_NOT_DISABLEABLE`).
- With a module disabled, its endpoints (`/api/v1/modules/<slug>/…`) return **403** and its UI disappears from the menu.

See [Managing modules](gestion.md).

## Dependencies

A manifest declares **hard** dependencies (`dependencies.modules` — without them the module does not start) and **soft** ones (`optionalModules` — documenting optional integrations). The registry orders loading topologically and detects cycles and incompatible versions. Disabling a module others depend on requires an explicit cascade (`force`).

## Next step

- [Catalog of the 24 modules](catalogo/index.md)
- [Managing modules in your installation](gestion.md)
- [Building a module](crear-un-modulo/index.md)
