# mod.ai-tutor — Tutor conversacional con RAG

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **ai** (desactivable)

## Qué hace

Tutor IA **por curso** con RAG: indexa el contenido al publicar el curso, genera embeddings (proveedor configurable por tenant vía el [AI Gateway](../../configuracion/ia.md)) y los guarda en pgvector; el chat vive en la página del curso y **cita las lecciones concretas** en las que se apoya cada respuesta.

Encima hay un **ciclo de calidad** en Administración → IA → Tutor con tres pestañas: revisión de pares pregunta→respuesta (con filtro «solo sin respaldo» para aislar respuestas sin citas), **conocimiento validado** escrito por humanos, e informe mensual con las preguntas agrupadas por tema y ordenadas por volumen.

## Cómo funciona

- Indexa al recibir `courses.course.published` y **reindexa por lección** al crear/editar/borrar lecciones — no hace falta reindexar el curso entero al subir una clase.
- Las **correcciones humanas viven en tabla propia**, no como chunks: se recuperan por similitud, se inyectan en el prompt **por encima** del contexto RAG y entran en caliente sin reindexar (y sobreviven a los reindexados, que borran y regeneran los chunks).
- Cuota de **10 preguntas diarias** por alumno (no aplica al staff, que además puede probar el tutor de cualquier curso sin matricularse).
- Cuotas superadas → 429; proveedor de IA sin configurar → 424.

## Dependencias

Opcionales: `mod.courses`, `mod.learning`.

## Modelo de datos

`mod_ai_tutor_conversation` · `_message` (con embedding de la pregunta y estado de revisión) · `_chunk` (trozos indexados, vector 1536) · `_correction` (conocimiento validado, vector 1536) · `_token_usage` (consumo para la cuota).

## API

`POST /modules/ai-tutor/courses/:courseId/ask` (alumno) + superficie admin (`/admin/ai-tutor/*`: indexado, revisión, correcciones, informe). Detalle en [Referencia → Pagos, aula e IA](../../api/referencia/pagos-directo-ia.md#ia).

## Eventos

- **Emite**: `ai-tutor.course.indexed/index-failed`, `ai-tutor.chat.message-sent/message-received`, `ai-tutor.answer.reviewed/corrected`.
- **Consume**: `courses.course.published/unpublished`, `courses.lesson.created/updated/deleted`.

## Configuración

Proveedor y clave en Administración → IA (requiere `AI_CONFIG_ENCRYPTION_KEY`). Los parámetros del RAG (top-K 5, presupuesto de historial 3000 tokens, umbral de correcciones, cuota diaria 10) son constantes del producto.
