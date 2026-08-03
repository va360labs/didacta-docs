# mod.access-groups — Grupos de acceso

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo — intentar desactivarlo responde `422 CORE_MODULE_NOT_DISABLEABLE`)

## Qué hace

Encapsula el *entitlement* «qué cursos puede ver un miembro» como pieza componible. Un grupo se define con tres tipos (`kind`):

- **`ALL_COURSES`** — todos los cursos publicados, con `autoGrantNewCourses` para matricular automáticamente en cada curso nuevo.
- **`COURSE`** / **`MULTI_COURSE`** — un set explícito de cursos.

La pertenencia al grupo se materializa como **matrículas reales** del core (vía `mod.learning`, con origen `GROUP`), nunca tocando tablas ajenas.

## Cómo funciona

- Los miembros entran por tres vías con dueño distinto: **MANUAL** (alta del admin o aprobación de inscripción — *sticky*, nunca se revoca sola), **TIER** (reconciliada con los tiers de `mod.payment-connections`) y **MEMBERSHIP** (concedida por la membresía de pago de `mod.subscriptions`).
- La clave del diseño es el **refcount con provenance**: un curso solo se desmatricula cuando ningún grupo vivo lo otorga, y jamás se tocan matrículas de origen `PURCHASE`, `SUBSCRIPTION` o `API`.
- La revocación por cancelación o impago retira **solo** las membresías de origen `MEMBERSHIP`. En `/admin/grupos-acceso` y en el dossier del usuario, esas membresías se distinguen con el badge «Por membresía».
- Borrar un grupo revoca sus membresías y limpia los calendarios de drip huérfanos.
- Un grupo puede marcarse `isDefaultForApproval` (se concede al aprobar inscripciones) o vincularse a un tier (`linkedTierName`).

Es un módulo first-party **formalizado como paquete** (`modules/access-groups/`, manifest propio) cuya orquestación NestJS —controller, service Prisma y bridges de eventos— vive en el host (`apps/api/src/modules/access-groups/`), el patrón estándar de módulo built-in.

## Dependencias

- Duras: `mod.courses`, `mod.learning`. Opcionales: `mod.payment-connections`, `mod.subscriptions`.

## Modelo de datos

`mod_access_groups_group` (grupo, kind, flags) · `mod_access_groups_group_course` (cursos del grupo) · `mod_access_groups_group_member` (membresía con `status` y `source`) · `mod_access_groups_grant` (provenance/refcount por grupo, usuario y curso).

## API

Prefijo `/modules/access-groups` — **todo requiere admin**, incluidas las lecturas (exponen el roster). Detalle en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#grupos-de-acceso-modulesaccess-groups-todo-admin).

## Eventos

- **Emite**: ninguno.
- **Consume**: `courses.course.published`, `payment_connections.user_tier.changed`, `subscriptions.membership.activated`, `subscriptions.subscription.activated/canceled/unpaid`.

## Configuración

Sin variables propias. Los dos puntos de configuración viven en sus dueños: `isDefaultForApproval` en el propio grupo y el grupo de la membresía en la configuración de `mod.subscriptions`.
