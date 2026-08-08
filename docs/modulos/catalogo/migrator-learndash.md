# mod.migrator-learndash — Migrador desde WordPress + LearnDash

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **migration** · **Beta** · Se distribuye como **ZIP firmado** (formato third-party)

## Qué hace

Migra una academia completa de **WordPress + LearnDash** a Didacta: cursos, lecciones, temas, quizzes, preguntas, usuarios, grupos, matrículas, media y progreso. Se opera con un **wizard de 6 pasos** (bienvenida → conectar con URL + Application Password → resumen del origen → opciones → comprobación previa *dry-run* → migración real con progreso en vivo), pensado para que un administrador no técnico la complete en una hora.

## Cómo funciona

- Pipeline **ETL con staging**: extrae a tablas de staging, transforma y carga con **mapeo idempotente** por checksum — relanzar no duplica.
- Los errores de datos van a una **DLQ** sin abortar el job; al final hay reportes de reconciliación (JSON/CSV/PDF firmado para auditor) y una cadena de auditoría SHA-256 verificable.
- **Solo lee** del origen: el WordPress queda intacto.
- Contraseñas: por defecto no importa hashes (PHPass no es compatible con Argon2id); cada usuario recibe email de activación para definir la suya.
- El job corre en backend (se puede cerrar el navegador), con timeout de 6 horas.
- Es el **ejemplo real del formato third-party**: se instala por ZIP en Administración → Marketplace módulos, corre en la VM aislada con `ctx.db`/`ctx.http`/`ctx.secrets` sandboxeados, y sus credenciales de WordPress se guardan cifradas AES-256-GCM con clave scoped al job.

## Dependencias

Requeridos: `mod.courses`, `mod.learning`, `mod.assessments`. Opcionales: `mod.certificates`, `mod.community`. Si falta un requerido, el wizard bloquea el inicio con aviso claro.

## Modelo de datos

13 tablas `mod_migrator_learndash_*`: `_jobs`, `_mappings` (origen ↔ destino por entidad), `_dlq`, `_audit_events` (append-only encadenada), `_validation_reports` y 10 de staging (`_stg_users`, `_stg_courses`, `_stg_lessons`, `_stg_quizzes`…).

## API

Prefijo `/modules/migrator-learndash`: `preflight`, `jobs` (crear, estado, progreso SSE, cancelar), reportes y verificación de auditoría.

## Configuración

Sin variables de entorno: todo viaja en el job que crea el wizard.

!!! warning "Límites del MVP"
    El rollback automático existe pero no está validado para producción — el camino soportado es restaurar desde el `pg_dump` previo. No se migra el histórico fino de intentos de quiz ni certificados ya emitidos (solo plantillas).
