# Cómo funcionan los módulos

Toda la funcionalidad de Didacta más allá del núcleo — cursos, evaluaciones, certificados, comunidad, pagos, aula virtual… — vive en **módulos**. El core aporta la plataforma (tenancy, auth, storage, eventos, auditoría); los módulos aportan el producto.

!!! info "Los módulos son siempre Community"
    Ningún módulo se bloquea por licencia, nunca. Lo único de pago en Didacta son [capabilities transversales del core](../enterprise/capabilities.md). Es una regla de diseño innegociable del proyecto.

## Dos niveles de módulo

Didacta distingue dos niveles con contratos distintos:

| | **First-party (in-tree)** | **Third-party (marketplace)** |
| --- | --- | --- |
| Dónde vive | En el repositorio, en `modules/<slug>/` | En un **ZIP firmado** que sube un `super_admin` |
| Confianza | Código del producto, revisado en el repo | Código externo, ejecutado en **sandbox** (VM aislada) |
| Acceso a datos | Prisma del host; puede **leer** (nunca escribir) tablas de otros módulos declarando la dependencia | Solo sus propias tablas, a través de un `db` sandboxed con guardia SQL por prefijo |
| UI | In-tree, en la web de Didacta | Bundles JS dentro del ZIP |

Los 24 módulos que trae Didacta son first-party. El nivel third-party existe para el futuro marketplace — ver [Módulos de terceros](modulos-de-terceros.md).

## Reglas de aislamiento

Todo módulo, del nivel que sea, cumple el mismo contrato:

- **Tablas propias** con prefijo `mod_<slug>_` y columna `tenant_id` + política RLS (automática — ver [Base de datos](../configuracion/base-de-datos.md)).
- **Cero foreign keys** entre tablas de módulos distintos.
- **Cero imports** de código privado de otro módulo.
- La comunicación entre módulos pasa por **eventos de dominio**, **hooks** o **APIs públicas** — y todo evento emitido o consumido se declara en el manifest.
- Un módulo first-party puede **leer** tablas de otro módulo si declara la dependencia en su manifest y filtra siempre por `tenant_id`. Escribir, jamás.

## El contexto de módulo

Un módulo no accede a la infraestructura directamente: recibe un `ModuleContext` con los servicios del core —

`eventBus` (eventos de dominio con outbox transaccional) · `hookRegistry` (puntos de extensión síncronos) · `storage` · `auditLog` · `evidenceVault` · `notificationHub` (email / in-app / webhook respetando preferencias) · `i18n` · `logger` · `config` (configuración por tenant y módulo).

## Activación por tenant

Los módulos se activan y desactivan **por organización**:

- El estado vive en la base de datos (`tenant_module`); si no hay registro, aplica el `enabledByDefault` del módulo.
- Los módulos de categoría **core** (cursos, aprendizaje, evaluaciones, certificados…) no se pueden desactivar.
- Con un módulo desactivado, sus endpoints (`/api/v1/modules/<slug>/…`) responden **403** y su UI desaparece del menú.

Ver [Gestionar módulos](gestion.md).

## Dependencias

Un manifest declara dependencias **duras** (`dependencies.modules` — sin ellas el módulo no arranca) y **blandas** (`optionalModules` — documentan integraciones opcionales). El registro ordena la carga topológicamente y detecta ciclos y versiones incompatibles. Desactivar un módulo del que dependen otros exige cascada explícita (`force`).

## Siguiente paso

- [Catálogo de los 24 módulos](catalogo.md)
- [Gestionar módulos en tu instalación](gestion.md)
- [Crear un módulo](crear-un-modulo/index.md)
