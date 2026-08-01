# Módulos de terceros (ZIP firmado)

Además de los módulos incluidos, Didacta puede ejecutar módulos **empaquetados como ZIP firmado** que un `super_admin` instala desde **Administración → Marketplace**. Es la vía prevista para el futuro marketplace de módulos.

!!! note "Estado: funcional, en evolución"
    El pipeline de instalación, el sandbox y el enrutado están implementados y operativos. El **catálogo público de marketplace aún no existe** (hoy la instalación es por subida manual del ZIP), los módulos de vendor `community` todavía no se aceptan (solo paquetes firmados por Didacta o subida directa consciente), y la desinstalación no revierte migraciones ni borra tablas.

## El formato del paquete

```
<modulo>.zip                          (máx. 50 MB)
├── manifest.jwt                      # manifest firmado (JWS compact, ES256)
├── package.json                      # { name, version, main: "dist/index.js" }
├── dist/index.js                     # bundle CommonJS del backend
├── dist/ui/<surface>.js              # bundles UI por superficie (admin, alumno…)
└── prisma/migrations/<ts>_<x>.sql    # migraciones SQL planas
```

- `manifest.jwt` es un JWT **ES256** cuyo payload es el manifest completo del módulo (nombre, versión, tablas, permisos, eventos, superficies UI, límites de red/BD…).
- La firma se verifica contra las claves públicas de Didacta; también se admite **subida directa sin firma** para paquetes propios, marcada explícitamente como tal.

## Qué valida la instalación

1. Tamaño, estructura del ZIP y seguridad de rutas (sin `..`, sin rutas absolutas).
2. Firma del manifest y coherencia interna: `tablePrefix` y `apiNamespace` deben derivar del nombre (`mod.mi-modulo` → `mod_mi_modulo_` y `/modules/mi-modulo`).
3. Nombre no reservado (no puede suplantar a un módulo built-in) y compatibilidad con la versión del core.
4. **Lint de las migraciones SQL**: solo pueden tocar tablas con su prefijo; nada de `CREATE FUNCTION`, `GRANT`, `TRUNCATE`, `COPY`, roles ni schemas.
5. **Lint estático del bundle**: sin acceso a `fs`, red directa, procesos ni `eval`; solo un conjunto cerrado de imports permitidos (`@didacta/core-kernel`, `zod`, criptografía y utilidades puras).

## El sandbox en ejecución

El código third-party corre en una **VM aislada de Node** con recursos *scoped* que le inyecta el host:

| Recurso | Qué permite |
| --- | --- |
| `ctx.db` | SQL **solo sobre sus tablas** (`mod_<slug>_*`), con prepared statements, timeout máx. 10 s, tope de 10 000 filas y sin DDL en runtime. |
| `ctx.http` | Llamadas salientes solo a los hosts declarados en el manifest, con rate limit y protección SSRF. |
| `ctx.didacta` | API pública del core con matriz de permisos e idempotencia. |
| `ctx.secrets` | Almacén cifrado AES-256-GCM, aislado por tenant y módulo. |
| `ctx.jobs` | Trabajos periódicos (`onJobTick`) sobre BullMQ. |

Sus rutas HTTP se publican bajo `/api/v1/modules/<slug>/…` y su UI se sirve como bundles por superficie (`admin`, `alumno`…).

!!! warning "La UI de terceros no está aislada"
    Los bundles de UI se ejecutan en el mismo contexto del navegador que la web de Didacta (no hay sandbox de UI todavía). Instala solo paquetes en los que confíes.

## Firmar tus propios paquetes

El empaquetado y firma oficial se hace con la tooling de VA360 LABS (`scripts/marketplace/sign-package.ts` contra el emisor de firmas). Para módulos internos de tu organización puedes usar la subida directa sin firma, asumiendo la advertencia de origen.

## Referencia

El ejemplo real completo de módulo third-party es `mod.migrator-learndash` (vive en el repo y se empaqueta con este formato). El contrato del paquete está en `packages/module-package-spec`.
