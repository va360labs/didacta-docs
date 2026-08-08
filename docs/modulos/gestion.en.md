# Managing modules

Modules are enabled and disabled **per organization (tenant)**, from the admin panel or over the API.

## From the panel

**Settings → Modules** lists the available modules with their state, description and dependencies. A `tenant_admin` manages the ones in their own organization; a `super_admin` can manage any organization's.

You already chose an initial set in the [setup wizard](../instalacion/setup-wizard.md); you can change it here at any time.

## Over the API

```bash
# List modules and their state
GET /api/v1/admin/modules

# Enable (idempotent)
POST /api/v1/admin/modules/mod.gamification/enable

# Disable
POST /api/v1/admin/modules/mod.gamification/disable

# Disable with cascade (if other active modules depend on it)
POST /api/v1/admin/modules/mod.courses/disable?force=true
```

## Rules

- **Core modules cannot be disabled.** Courses, learning, assessments, certificates, access groups, registration, billing, subscriptions… return **422** `CORE_MODULE_NOT_DISABLEABLE`.
- **Dependencies**: if you try to disable a module other active modules depend on, the API returns `MODULE_HAS_ACTIVE_DEPENDENTS`; with `force=true` they are disabled in cascade.
- **Data is preserved**: disabling a module does not delete its tables. Re-enable it and everything is exactly where it was.
- Every change is recorded in the **audit log** (`admin.module.enabled` / `admin.module.disabled`) and publishes a domain event.

## What disabling a module does

- Its endpoints (`/api/v1/modules/<slug>/…`) return **403** ("module X is not active for this tenant"), with a state cache of about 30 seconds.
- Its menu entries and pages disappear from the web app for that organization's users.
- Its background workers check the state before acting on the tenant's data.

## What the end user sees

The frontend calls `GET /api/v1/me/modules`, which returns the active modules of the user's organization (plus the instance's active Enterprise capabilities). The sidebar is built from that response — there are no "dead" menu entries.

## Third-party modules

Installing external modules (a signed ZIP) is an **instance-level** operation reserved for `super_admin`, and it is managed under **Administration → Module marketplace**. See [Third-party modules](modulos-de-terceros.md).
