# mod.courses — Cursos y catálogo

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Es el corazón del catálogo: cursos con `slug`, título y estado (`DRAFT` → `PUBLISHED` → `ARCHIVED`), **módulos** como agrupación ordenada dentro del curso y **lecciones** tipadas (`VIDEO`, `HTML`, `PDF`, `TEXT`, `QUIZ`, `SCORM`) con contenido JSON flexible. Incluye categorías curadas por el administrador (nombre, color, icono) y operaciones de edición avanzadas: reordenar módulos y lecciones en bloque (drag & drop) y mover lecciones entre módulos.

## Cómo funciona

- La **publicación pasa por un hook abierto**: antes de publicar, `mod.courses` ejecuta `courses.publish.validate` y cualquier módulo registrado puede añadir razones de bloqueo (por ejemplo, `mod.fundae` exige objetivos y duración). Si hay razones, la publicación falla con 422 y la lista de motivos.
- Los borrados son **soft-delete** con cascada lógica: eliminar un módulo o lección preserva el progreso histórico de los alumnos.
- Es la pieza de la que cuelgan casi todos los demás módulos (learning, access-groups, billing, assessments, ai-content, ai-tutor…).

## Dependencias

- Duras: ninguna.
- Opcional: `mod.certificates` (valida la plantilla de certificado asignada al curso).

## Modelo de datos

| Tabla | Qué guarda |
| --- | --- |
| `mod_courses_course` | Curso: slug, título, estado, categoría, plantilla de certificado. |
| `mod_courses_module` | Agrupación ordenada dentro del curso. |
| `mod_courses_lesson` | Lección: tipo, contenido JSONB, duración, fecha de publicación programada. |
| `mod_courses_category` | Categorías curadas (nombre, color, icono). |

## API

Prefijo `/modules/courses`: CRUD de cursos/módulos/lecciones, categorías gestionadas, transiciones de estado y bloque de ordenación. Detalle en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#cursos-modulescourses).

## Eventos y hooks

- **Emite**: `courses.course.created/updated/published/archived/unarchived`, `courses.module.created/deleted`, `courses.lesson.created/updated/moved`.
- **Expone el hook** `courses.publish.validate` (el único hook activo del producto — el patrón de referencia).

## Configuración

Sin ajustes por tenant ni variables de entorno propias.
