# mod.access-groups — Access groups

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active — trying to disable it returns `422 CORE_MODULE_NOT_DISABLEABLE`)

## What it does

It encapsulates the entitlement "which courses a member can see" as a composable building block. A group is defined with one of three kinds (`kind`):

- **`ALL_COURSES`** — every published course, with `autoGrantNewCourses` to enroll members automatically in each new course.
- **`COURSE`** / **`MULTI_COURSE`** — an explicit set of courses.

Group membership materialises as **real core enrollments** (through `mod.learning`, with origin `GROUP`), never by touching another module's tables.

## How it works

- Members arrive through three routes with different owners: **MANUAL** (added by an admin or on registration approval — *sticky*, never revoked on its own), **TIER** (reconciled against `mod.payment-connections` tiers) and **MEMBERSHIP** (granted by the paid membership from `mod.subscriptions`).
- The key design idea is **refcounting with provenance**: a course is only unenrolled when no live group grants it, and enrollments with origin `PURCHASE`, `SUBSCRIPTION` or `API` are never touched.
- Revocation on cancellation or non-payment removes **only** the memberships whose origin is `MEMBERSHIP`. In `/admin/grupos-acceso` and in the user dossier, those memberships are marked with the "By membership" badge.
- Deleting a group revokes its memberships and cleans up orphaned drip schedules.
- A group can be flagged `isDefaultForApproval` (granted when registrations are approved) or linked to a tier (`linkedTierName`).

It is a first-party module **formalised as a package** (`modules/access-groups/`, with its own manifest) whose NestJS orchestration — controller, Prisma service and event bridges — lives in the host (`apps/api/src/modules/access-groups/`), the standard built-in module pattern.

## Dependencies

- Hard: `mod.courses`, `mod.learning`. Optional: `mod.payment-connections`, `mod.subscriptions`.

## Data model

`mod_access_groups_group` (group, kind, flags) · `mod_access_groups_group_course` (the group's courses) · `mod_access_groups_group_member` (membership with `status` and `source`) · `mod_access_groups_grant` (provenance/refcount per group, user and course).

## API

Prefix `/modules/access-groups` — **everything requires admin**, reads included (they expose the roster). Details in [Reference → Learning](../../api/referencia/aprendizaje.md#access-groups-modulesaccess-groups-all-admin).

## Events

- **Emits**: none.
- **Consumes**: `courses.course.published`, `payment_connections.user_tier.changed`, `subscriptions.membership.activated`, `subscriptions.subscription.activated/canceled/unpaid`.

## Configuration

No variables of its own. The two configuration points live with their owners: `isDefaultForApproval` on the group itself, and the membership's group in the `mod.subscriptions` configuration.
