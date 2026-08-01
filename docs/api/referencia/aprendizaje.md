# Referencia — Aprendizaje

Endpoints de cursos, matrículas y progreso, rutas, evaluaciones, certificados, grupos de acceso, grupos, eventos y Fundae. Todas las rutas cuelgan de `/api/v1`.

**Leyenda de Auth**: `Bearer` = cualquier usuario autenticado del tenant · `formador+` = formador, tenant_admin o super_admin · `admin` = tenant_admin o super_admin · `Público` = sin sesión.

## Cursos — `/modules/courses`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/courses` | Bearer | Lista cursos del tenant; query `status`, `q`, `category`. |
| POST | `/modules/courses` | Bearer | Crea curso en DRAFT. |
| GET | `/modules/courses/categories` | Bearer | Categorías usadas por cursos publicados. |
| GET | `/modules/courses/managed-categories` | Bearer | Categorías curadas del tenant (color, icono). |
| POST | `/modules/courses/managed-categories` | admin | Crea categoría curada. |
| PUT | `/modules/courses/managed-categories/:id` | admin | Actualiza categoría curada. |
| DELETE | `/modules/courses/managed-categories/:id` | admin | Borra categoría curada. |
| GET | `/modules/courses/:id` | Bearer | Detalle con módulos y lecciones (ver gating abajo). |
| PUT | `/modules/courses/:id` | Bearer | Actualiza metadatos del curso. |
| POST | `/modules/courses/:id/modules` | Bearer | Añade módulo al curso. |
| POST | `/modules/courses/modules/:moduleId/lessons` | Bearer | Añade lección al módulo. |
| PUT | `/modules/courses/lessons/:lessonId` | Bearer | Actualiza contenido de una lección. |
| POST | `/modules/courses/:id/publish` | Bearer | Publica (ejecuta el hook `courses.publish.validate`). |
| POST | `/modules/courses/:id/archive` · `/:id/unarchive` | Bearer | Archiva / vuelve a DRAFT. |
| POST | `/modules/courses/lessons/:lessonId/move` | Bearer | Mueve la lección un puesto arriba/abajo. |
| POST | `/modules/courses/lessons/:lessonId/move-to-module` | Bearer | Mueve la lección a otro módulo. |
| POST | `/modules/courses/modules/:moduleId/reorder-lessons` | Bearer | Reordena lecciones en bloque (drag & drop). |
| POST | `/modules/courses/:id/reorder-modules` | Bearer | Reordena módulos del curso en bloque. |
| DELETE | `/modules/courses/modules/:moduleId` | Bearer | Soft-delete del módulo (cascade lógico de lecciones). |
| DELETE | `/modules/courses/lessons/:lessonId` | Bearer | Soft-delete de la lección (preserva progreso histórico). |

**Bodies clave** — crear curso: `slug` (kebab-case), `title`, `description?`, `thumbnailUrl?`, `language` (default `es-ES`), `estimatedMinutes?`, `category?`. `slug` y `language` son inmutables. Crear lección: `type` (`VIDEO|HTML|PDF|TEXT|QUIZ|SCORM`), `title`, `content` (objeto libre), `durationMinutes?`, `publishAt?` (fecha futura = bloqueada).

**Gating de lectura de `GET /:id`**: formador+ recibe el curso completo; el alumno recibe 404 si no está `PUBLISHED`, la estructura con `content: null` si no está matriculado, y `content: null` solo en lecciones no liberadas si hay drip.

**Errores**: `COURSE_NOT_FOUND` 404 · `COURSE_SLUG_EXISTS` 409 · `COURSE_ALREADY_PUBLISHED` 409 · `COURSE_NO_LESSONS` 422 · `COURSE_PUBLISH_VALIDATION_FAILED` 422 (con array `reasons`).

## Matrículas, progreso y drip — `/modules/learning`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/learning/me/enrollments` | Bearer | Mis matriculaciones. |
| GET | `/modules/learning/me/stats` | Bearer | Mis estadísticas (cursos completados, tiempo visto). |
| GET | `/modules/learning/me/enrollments/:id/progress` | Bearer | Mi progreso por lección de una matrícula. |
| POST | `/modules/learning/enrollments/me` | Bearer | Auto-matriculación en un curso. |
| POST | `/modules/learning/enrollments/by-code` · `by-link` | Bearer | Auto-matriculación con código / token de invitación. |
| POST | `/modules/learning/enrollments` | formador+ | Matricula a otro usuario (origen ADMIN). |
| DELETE | `/modules/learning/enrollments/:id` | Bearer | Cancela mi matrícula. |
| DELETE | `/modules/learning/enrollments/:id/by-admin` | formador+ | Baja de la matrícula de un alumno. |
| POST | `/modules/learning/progress` | Bearer | Reporta progreso: `{ enrollmentId, lessonId, watchedSeconds, resumePositionSec?, completed? }`. |
| GET | `/modules/learning/courses/:courseId/enrollments` | formador+ | Alumnos matriculados en el curso. |
| GET | `/modules/learning/courses/:courseId/enrollments/:id/progress` | formador+ | Progreso detallado de un alumno. |
| GET | `/modules/learning/invitations` | Bearer | Invitaciones activas de un curso (`?courseId=`). |
| POST | `/modules/learning/invitations` | Bearer | Crea invitación (`courseId`, `maxUses?`, `expiresAt?`) → código + token. |
| DELETE | `/modules/learning/invitations/:id` | Bearer | Revoca invitación. |
| GET | `/modules/learning/courses/:courseId/drip` | formador+ | Calendarios de drip del curso. |
| POST | `/modules/learning/courses/:courseId/drip` | formador+ | Crea calendario: `audienceKind` (`TIER\|GROUP`), `audienceRef`, `unit` (`LESSON\|MODULE`), `intervalDays` (≥1), `startOffsetDays?`. |
| PUT | `/modules/learning/drip/:id` · DELETE | formador+ | Edita / borra calendario de drip. |
| GET | `/modules/learning/courses/:courseId/availability` | Bearer | Fechas de desbloqueo de las lecciones para el alumno actual. |
| GET/POST/DELETE | `/modules/learning/lessons/:lessonId/unlock-subscription` | Bearer | Consulta / alta / baja del aviso por email de desbloqueo. |
| GET | `/modules/learning/lessons/:lessonId/comments` | Bearer | Comentarios (APPROVED de todos + propios; formador+ ve PENDING). |
| POST | `/modules/learning/lessons/:lessonId/comments` | Bearer | Crea comentario (nace PENDING hasta moderación). |
| GET | `/modules/learning/courses/:courseId/comments/pending` | formador+ | Cola de moderación del curso. |
| POST | `/modules/learning/comments/:id/approve` · `reject` | formador+ | Modera un comentario (`reject` admite `reason?`). |
| DELETE | `/modules/learning/comments/:id` | Bearer (autor) | Borra el comentario propio. |
| GET | `/modules/learning/me/competencies` · `/modules/learning/competencies` | Bearer | Mi mapa de competencias / catálogo del tenant. |
| POST · DELETE | `/modules/learning/competencies[/:id]` | formador+ | Crea / elimina competencia. |
| GET · PUT | `/modules/learning/courses/:courseId/competencies` | Bearer · formador+ | Competencias del curso / reemplaza el set (`items: [{competencyId, weight?}]`). |
| POST | `/modules/learning/lessons/:lessonId/scorm` | formador+ | Sube paquete SCORM 1.2/2004 en base64 (máx. ~100 MiB binarios). |
| GET | `/modules/learning/lessons/:lessonId/scorm` | Bearer | Metadata + URL firmada del entry para el iframe (exige matrícula activa salvo editores). |
| POST | `/modules/learning/lessons/:lessonId/scorm/attempt` · `commit` | Bearer | Inicia/reanuda el intento SCORM · persiste el estado `cmi` (al completar, puentea al progreso). |
| POST | `/modules/learning/lesson-unlock/run-now` | super_admin | Fuerza un ciclo del notificador de desbloqueo (QA). |

**Errores**: `ALREADY_ENROLLED` 409 · `ENROLLMENT_NOT_FOUND` 404 · `INVITATION_INVALID` 400 · `COURSE_NOT_PUBLISHED` 422 · `LESSON_LOCKED` 403 · `TRIAL_CONTENT_LOCKED` 403 (contenido de prueba, se desbloquea pagando) · `SCORM_*` 400/404/422.

## Rutas de aprendizaje — `/modules/learning/paths`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/learning/paths` | Bearer | Rutas publicadas con mi progreso. |
| GET | `/modules/learning/me/paths` | Bearer | Mis rutas (activas + completadas). |
| GET | `/modules/learning/paths/formador` | formador+ | Todas las rutas en cualquier estado (panel formador). |
| GET | `/modules/learning/paths/:slug` | Bearer | Detalle de una ruta publicada con sus cursos. |
| POST | `/modules/learning/paths` | formador+ | Crea ruta: `title`, `description?`, `sequenceType?` (`LINEAR\|FLEXIBLE`). |
| PATCH | `/modules/learning/paths/:id` | formador+ | Actualiza (incluye `courses: [{courseId, position}]` — reemplaza el set). |
| POST | `/modules/learning/paths/:id/publish` · `archive` · `restore` | formador+ | Publica/despublica · archiva · restaura a DRAFT. |
| POST · DELETE | `/modules/learning/paths/:id/enroll` | Bearer | Matrícula en la ruta (y sus cursos) · cancelación. |

**Errores**: ruta no encontrada / no publicada 404 · ya matriculado 409 · ruta sin cursos 400.

## Evaluaciones — `/modules/assessments`

**Gestión (formador+):**

| Método | Ruta | Qué hace |
|---|---|---|
| POST | `/modules/assessments/quizzes` | Crea quiz en DRAFT: `title`, `lessonId?`, `passThreshold?` (0-100), `maxAttempts?`, `timeLimitMinutes?`, `shuffleQuestions?`, `showFeedback?`. |
| GET · PUT | `/modules/assessments/quizzes/:id` | Detalle para el formador (incluye `isCorrect`) · actualización. |
| POST | `/modules/assessments/quizzes/:id/questions` | Añade pregunta: `type` (`SINGLE_CHOICE\|MULTIPLE_CHOICE\|TRUE_FALSE\|FILL_IN_BLANK\|SHORT_ANSWER\|LONG_ANSWER`), `prompt`, `options?`, `acceptedAnswers?`, `points?`. |
| DELETE | `/modules/assessments/quizzes/:id/questions/:questionId` | Soft-delete de la pregunta. |
| POST | `/modules/assessments/quizzes/:id/publish` | Publica (exige ≥1 pregunta). |
| GET | `/modules/assessments/attempts/pending` | Intentos en `PENDING_REVIEW` (corrección manual de respuestas abiertas). |
| GET | `/modules/assessments/attempts/:id/full` | Intento completo para el corrector. |
| POST | `/modules/assessments/attempts/:id/grade` | Califica manualmente: `{ grades: [{questionId, scoreEarned, feedback?}] }`; emite `assessments.attempt.passed/failed`. |

**Alumno (Bearer):**

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/assessments/quizzes/:id/preview` | Vista del quiz **sin** soluciones. |
| POST | `/modules/assessments/attempts` | Inicia intento: `{ quizId, enrollmentId?, lessonId? }`. |
| POST | `/modules/assessments/attempts/:id/submit` | Envía respuestas `{ answers: [{questionId, selectedOptionIds?, textAnswer?}] }`; autocorrección + eventos. |
| GET | `/modules/assessments/attempts/:id` | Detalle de un intento propio. |
| GET | `/modules/assessments/attempts?quizId=` | Mis intentos de un quiz. |

**Errores**: `QUIZ_NOT_PUBLISHED` / `QUIZ_HAS_NO_QUESTIONS` 422 · `ATTEMPT_ALREADY_SUBMITTED` / `MAX_ATTEMPTS_REACHED` 409 · `ATTEMPT_EXPIRED` **410** · not-found 404.

## Certificados — `/modules/certificates`

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/certificates/me` | Bearer | Mis certificados emitidos. |
| GET | `/modules/certificates/:id` · `/:id/download` | Bearer | Detalle · descarga del PDF (regenerado desde snapshot inmutable). |
| GET | `/modules/certificates/verify/:id` | **Público** | Verificación pública: `{ number, studentName, courseTitle, issuedAt, valid }`. Nunca expone email ni datos internos. |
| GET · POST | `/modules/certificates/templates` | formador+ | Lista · crea plantilla: `name`, `body`, `primaryColor?`, `logoUrl?`, `signerName?`, `signerTitle?`, `isDefault?`. |
| GET · PATCH · DELETE | `/modules/certificates/templates/:id` | formador+ | Detalle · edición · borrado (409 si es default o está en uso). |
| POST | `/modules/certificates/templates/preview` | formador+ | PDF de previsualización con datos dummy, sin persistir. |
| POST | `/modules/certificates/templates/:id/set-default` | formador+ | Marca como plantilla por defecto del tenant. |

## Grupos de acceso — `/modules/access-groups` (todo admin)

| Método | Ruta | Qué hace |
|---|---|---|
| GET | `/modules/access-groups` | Lista paginada (`page`, `limit`). |
| GET | `/modules/access-groups/catalog/courses` · `catalog/users` | Selectores: cursos publicados · usuarios candidatos (`?q=`). |
| GET | `/modules/access-groups/:id` | Detalle con cursos y miembros. |
| POST | `/modules/access-groups` | Crea: `name`, `slug?`, `kind` (`ALL_COURSES\|COURSE\|MULTI_COURSE`), `courseIds?`, `autoGrantNewCourses?`. |
| PATCH | `/modules/access-groups/:id` | Edita (`name`, `description`, `autoGrantNewCourses`, `isDefaultForApproval`, `linkedTierName` — vincula un tier de pagos). |
| PUT | `/modules/access-groups/:id/courses` | Reemplaza el set completo de cursos. |
| POST | `/modules/access-groups/:id/members` | Asigna miembros: `{ userIds: [] }` (máx. 500). |
| DELETE | `/modules/access-groups/:id/members/:userId` | Revoca un miembro. |
| DELETE | `/modules/access-groups/:id` | Elimina el grupo y revoca sus membresías. |

## Grupos y eventos de comunidad

| Método | Ruta | Auth | Qué hace |
|---|---|---|---|
| GET | `/modules/groups` · `/me` · `/:id` | Bearer | Lista paginada · mis grupos · detalle con miembros. |
| POST | `/modules/groups` | formador+ | Crea grupo (`name`, `slug`, `description?`); el creador queda como owner. |
| POST · DELETE | `/modules/groups/:id/join` · `/:id/leave` | Bearer | Unirse · abandonar (idempotente). |
| GET | `/modules/events` · `/:id` | Bearer | Eventos por rango de fechas (`from`, `to`, `limit`, `order`) · detalle con `registeredCount`, `isFull`, `isRegistered`. |
| POST | `/modules/events` | formador+ | Crea evento: `title`, `startAt`, `endAt`, `location?`, `capacity?`. |
| POST | `/modules/events/:id/register` · `unregister` | Bearer | Inscripción (si está lleno: `{ registered: false, reason: 'full' }`) · cancelación. |

## Fundae — `/modules/fundae` (todo admin; el rol formador no accede)

| Método | Ruta | Qué hace |
|---|---|---|
| GET · POST | `/modules/fundae/actions` | Lista (filtros `courseId`, `status`) · crea acción formativa: `codigoAccion` (≤25), `nombre`, `modalidad` (`PRESENCIAL\|TELEFORMACION\|MIXTA`), `horasFormacion`, `fechaInicio`/`fechaFin` (`YYYY-MM-DD`), `courseId?`, `lugar?`, `cifCentro?`. |
| GET · PUT · DELETE | `/modules/fundae/actions/:id` | Detalle · actualización (+ `status`) · archivado (soft). |
| GET | `/modules/fundae/actions/:id/participants` · `/count` | Participantes con email, DNI, progreso y resultado · recuento. |
| GET | `/modules/fundae/actions/:id/export.xml` | XML Fundae de la acción (descarga). |
| GET | `/modules/fundae/actions/:id/participants/:userId/evidence.pdf` | PDF de evidencia firmada de un participante. |
| GET | `/modules/fundae/actions/:id/export.zip` | ZIP de presentación: XML + un PDF de evidencia por participante. |
| GET · POST | `/modules/fundae/actions/:id/blocks` | Módulos formativos (bloques) · alta (`ordinal?`, `title`, `hours`, `modalidad`, `contenidos?`). |
| PUT · DELETE | `/modules/fundae/actions/:id/blocks/:blockId` | Edición · borrado de bloque. |

La suma de horas de los bloques no puede superar las `horasFormacion` de la acción (Fundae lo verifica al subir el XML). El firmante de las evidencias es el administrador que dispara la descarga.
