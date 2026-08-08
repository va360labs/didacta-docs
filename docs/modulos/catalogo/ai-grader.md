# mod.ai-grader — Corrección IA con rúbrica

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **ai** (desactivable)

## Qué hace

Corrección asistida por IA para las preguntas abiertas de [mod.assessments](assessments.md) (`SHORT_ANSWER`, `LONG_ANSWER`). El formador define una **rúbrica por pregunta** (instrucciones + 1-8 criterios con peso); la IA lee la rúbrica y la respuesta del alumno y propone **nota con justificación por criterio** y feedback global. La nota **nunca se publica sola**: el formador revisa y aplica (o no) la sugerencia. El módulo registra métricas de acuerdo IA-humano.

## Cómo funciona

- La suma de los pesos de los criterios debe igualar los puntos de la pregunta (validado al guardar la rúbrica).
- Hay **una sugerencia por respuesta**: regenerar con `force` reemplaza la anterior; sin `force`, se sirve la cacheada sin volver a llamar al modelo.
- «Aplicar» una sugerencia solo la marca como aplicada para auditoría — la nota real se pone con la corrección manual de assessments (`POST attempts/:id/grade`), manteniendo un único camino de calificación.
- Las sugerencias se persisten para poder revisarlas después sin gastar tokens.

## Dependencias

Opcional: `mod.assessments` (de donde salen intentos, respuestas y preguntas).

## Modelo de datos

`mod_ai_grader_rubric` (una por pregunta: instrucciones + criterios JSON con peso) · `mod_ai_grader_suggestion` (nota propuesta, desglose por criterio, proveedor/modelo, trazas de aplicación).

## API

Prefijo `/modules/ai-grader` (formador+): gestión de rúbricas y ciclo de sugerencia. Detalle en [Referencia → Pagos, aula e IA](../../api/referencia/pagos-directo-ia.md#ia).

## Eventos

**Emite**: `ai-grader.suggestion.generated`, `ai-grader.suggestion.failed`. No consume.

## Configuración

Proveedor IA en Administración → Proveedores de IA; sin variables propias.
