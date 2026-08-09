# mod.courses — Courses and catalog

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

This is the heart of the catalog: courses with a `slug`, title and status (`DRAFT` → `PUBLISHED` → `ARCHIVED`), **modules** as an ordered grouping inside a course, and typed **lessons** (`VIDEO`, `HTML`, `PDF`, `TEXT`, `QUIZ`, `SCORM`) with flexible JSON content. It includes admin-curated categories (name, color, icon) and advanced editing operations: bulk reordering of modules and lessons (drag & drop) and moving lessons between modules.

## How it works

- **Publishing goes through an open hook**: before publishing, `mod.courses` runs `courses.publish.validate` and any registered module can add blocking reasons (for example, `mod.fundae` requires objectives and a duration). If there are any reasons, publishing fails with a 422 and the list of reasons.
- Deletions are **soft deletes** with a logical cascade: removing a module or a lesson preserves students' historical progress.
- It is the piece almost every other module hangs off (learning, access-groups, billing, assessments, ai-content, ai-tutor…).

## Dependencies

- Hard: none.
- Optional: `mod.certificates` (validates the certificate template assigned to the course).

## Data model

| Table | What it stores |
| --- | --- |
| `mod_courses_course` | The course: slug, title, status, category, certificate template. |
| `mod_courses_module` | An ordered grouping inside the course. |
| `mod_courses_lesson` | The lesson: type, JSONB content, duration, scheduled publication date. |
| `mod_courses_category` | Curated categories (name, color, icon). |

## API

Prefix `/modules/courses`: CRUD for courses/modules/lessons, managed categories, status transitions and the bulk ordering endpoint. Details in [Reference → Learning](../../api/referencia/aprendizaje.md#courses-modulescourses).

## Events and hooks

- **Emits**: `courses.course.created/updated/published/archived/unarchived`, `courses.module.created/deleted`, `courses.lesson.created/updated/moved`.
- **Exposes the hook** `courses.publish.validate` (the only active hook in the product — the reference pattern).

## Configuration

No per-tenant settings and no environment variables of its own.
