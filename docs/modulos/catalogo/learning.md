# mod.learning — Player, matrícula y progreso

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Gestiona todo el ciclo del alumno dentro de un curso: **matriculación** (directa, por admin, por código o enlace de invitación), **progreso por lección** (segundos vistos, posición de reanudación) y **finalización** con umbral configurable (75% por defecto). Añade además:

- **Drip content**: calendarios de liberación programada por intervalos relativos a la fecha de entrada del alumno, segmentables por tier de pago o por grupo de acceso, con aviso opcional por email al desbloquearse una lección.
- **Invitaciones** con código corto o token URL, con límite de usos, caducidad y revocación.
- **Comentarios de lección** con cola de moderación (nacen `PENDING` hasta que el formador aprueba).
- **Rutas de aprendizaje** (itinerarios de cursos, lineales o flexibles) y **competencias** ponderadas por curso.
- **SCORM 1.2/2004**: subida del paquete, parseo del `imsmanifest.xml` y persistencia del estado `cmi.*` por intento.

## Cómo funciona

- Solo se puede matricular en cursos `PUBLISHED` (`COURSE_NOT_PUBLISHED` 422); una matrícula activa duplicada responde `ALREADY_ENROLLED` 409.
- Cuando a un alumno le aplican varios calendarios de drip, **gana el más permisivo** (la fecha de desbloqueo más temprana). Sin calendario aplicable, todo está disponible.
- El contenido de prueba (trial de membresía) se gatea con `TRIAL_CONTENT_LOCKED` 403, que el frontend convierte en CTA de pago.
- Al cruzar el umbral emite `learning.course.completed` **una sola vez** — es lo que dispara la emisión de certificados.

## Dependencias

- Dura: `mod.courses`.
- Opcionales: `mod.payment-connections` (tier para el drip), `mod.access-groups` (grupos para el drip), `mod.subscriptions` (estado trial + límite de lecciones).

## Modelo de datos

`mod_learning_enrollment` (matrícula única por usuario y curso) · `mod_learning_progress` · `mod_learning_invitation` · `mod_learning_drip_schedule` · `mod_learning_lesson_unlock_sub` · `mod_learning_scorm_package` · `mod_learning_scorm_attempt` · `mod_learning_lesson_comment` · `mod_learning_path` + `_path_course` + `_path_enrollment` · `mod_learning_competency` + `_course_competency`.

## API

Prefijo `/modules/learning` (+ `/modules/learning/paths`). Detalle completo en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#matriculas-progreso-y-drip-moduleslearning).

## Eventos

- **Emite**: `learning.enrollment.created`, `learning.enrollment.cancelled`, `learning.progress.updated`, `learning.course.completed`, `learning.invitation.created`.
- **Consume**: `courses.course.archived`.

## Configuración

Sin variables propias: el umbral de finalización es por matrícula y el drip se configura por curso desde la API/panel. El cron del notificador de desbloqueo es `LESSON_UNLOCK_CRON`.
