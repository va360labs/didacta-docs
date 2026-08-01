# mod.assessments — Quizzes y exámenes

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **core** (siempre activo)

## Qué hace

Evaluaciones con autocorrección y corrección manual. Seis tipos de pregunta: `SINGLE_CHOICE`, `MULTIPLE_CHOICE` y `TRUE_FALSE` (corregidas automáticamente por un motor de scoring puro y testeable), `FILL_IN_BLANK` (respuestas aceptadas) y `SHORT_ANSWER`/`LONG_ANSWER` (derivan a **corrección manual del formador**, con bandeja de pendientes). Cada quiz configura umbral de aprobación (60% por defecto), máximo de intentos, límite de tiempo, barajado de preguntas y si se muestra feedback al terminar.

## Cómo funciona

- El quiz nace en `DRAFT`, se le añaden preguntas y se publica (exige al menos una).
- El alumno ve una **vista sin soluciones** (`preview`), inicia un intento, responde y entrega; el motor corrige lo automático y, si hay preguntas abiertas, el intento queda en `PENDING_REVIEW` hasta que el formador califica desde su panel.
- Un intento caducado por límite de tiempo responde `ATTEMPT_EXPIRED` **410**.
- La vista del formador incluye `isCorrect` en las opciones; la del alumno, jamás.
- Con [mod.ai-grader](ai-grader.md) activo, el formador recibe sugerencias de nota con justificación por criterio para las preguntas abiertas.

## Dependencias

- Dura: `mod.courses`. Opcional: `mod.learning` (autocompleta la lección de tipo QUIZ al aprobar).

## Modelo de datos

| Tabla | Qué guarda |
| --- | --- |
| `mod_assessments_quiz` | Quiz: umbral, intentos, tiempo, estado. |
| `mod_assessments_question` | Enunciado, tipo, puntos, respuestas aceptadas. |
| `mod_assessments_option` | Opciones de las preguntas de elección. |
| `mod_assessments_attempt` | Intento del alumno y su nota. |
| `mod_assessments_answer` | Respuesta por pregunta (opción o texto). |

## API

Prefijo `/modules/assessments`, dos superficies: autoría/corrección (formador+) y alumno. Detalle en [Referencia → Aprendizaje](../../api/referencia/aprendizaje.md#evaluaciones-modulesassessments).

## Eventos

**Emite**: `assessments.quiz.published`, `assessments.attempt.started/submitted/pending_review/graded/passed/failed`. No consume.

## Configuración

Sin ajustes por tenant ni variables propias: toda la configuración es por quiz.
