# Gestionar módulos

Los módulos se activan y desactivan **por organización (tenant)**, desde el panel de administración o por API.

## Desde el panel

**Administración → Configuración → Módulos** (la pestaña «Módulos» de `/admin/configuracion`) lista los módulos disponibles con su estado, descripción y dependencias. Un `tenant_admin` gestiona los de su organización; un `super_admin` puede gestionar los de cualquiera.

En el [asistente de configuración](../instalacion/setup-wizard.md) ya elegiste un conjunto inicial; puedes cambiarlo aquí en cualquier momento.

## Por API

```bash
# Listar módulos y su estado
GET /api/v1/admin/modules

# Activar (idempotente)
POST /api/v1/admin/modules/mod.gamification/enable

# Desactivar
POST /api/v1/admin/modules/mod.gamification/disable

# Desactivar en cascada (si otros módulos activos dependen de él)
POST /api/v1/admin/modules/mod.courses/disable?force=true
```

## Reglas

- **Los módulos core no se desactivan.** Cursos, aprendizaje, evaluaciones, certificados, grupos de acceso, inscripción, billing, suscripciones… devuelven **422** `CORE_MODULE_NOT_DISABLEABLE`.
- **Dependencias**: si intentas desactivar un módulo del que dependen otros activos, la API responde `MODULE_HAS_ACTIVE_DEPENDENTS`; con `force=true` se desactivan en cascada.
- **Los datos se conservan**: desactivar un módulo no borra sus tablas. Al reactivarlo, todo sigue donde estaba.
- Cada cambio queda en el **log de auditoría** (`admin.module.enabled` / `admin.module.disabled`) y publica un evento de dominio.

## Efecto de desactivar un módulo

- Sus endpoints (`/api/v1/modules/<slug>/…`) responden **403** («El módulo X no está activo para este tenant»), con una caché de estado de ~30 segundos.
- Sus entradas de menú y páginas desaparecen de la web para los usuarios de esa organización.
- Sus workers en segundo plano comprueban el estado antes de actuar sobre datos del tenant.

## Qué ve el usuario final

El frontend consulta `GET /api/v1/me/modules`, que devuelve los módulos activos de la organización del usuario (y las capabilities Enterprise activas de la instancia). El menú lateral se construye a partir de esa respuesta — no hay menús «muertos».

## Módulos de terceros

La instalación de módulos externos (ZIP firmado) es una operación de **instancia**, reservada a `super_admin`, y se gestiona en **Administración → Marketplace módulos**. Ver [Módulos de terceros](modulos-de-terceros.md).
