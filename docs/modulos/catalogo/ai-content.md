# mod.ai-content — Generador de contenido formativo

<span class="didacta-chip didacta-chip--community">Community</span> · Categoría **ai** (desactivable)

## Qué hace

Genera **borradores formativos** a partir del texto de una lección: resúmenes (`SUMMARY`), flashcards (`FLASHCARDS`) y quizzes (`QUIZ`). El resultado se persiste siempre como **borrador**: el formador revisa, puede editar el JSON, y después publica o rechaza con motivo. **Human-in-the-loop por defecto** — nada se publica automáticamente.

## Cómo funciona

- El prompt es específico por tipo y la llamada pasa por el [AI Gateway](../../configuracion/ia.md) con la configuración del tenant.
- Cada borrador guarda telemetría de la generación: proveedor, modelo y tokens de entrada/salida.
- Una lección sin texto responde `AI_CONTENT_LESSON_TEXT_EMPTY` 422; sin proveedor configurado, 503 con mensaje claro.
- Solo se puede editar/publicar/rechazar un borrador en estado `DRAFT` (409 en otro estado).
- El formador espera la generación (~5-30 s); no hay streaming todavía. La ingestión automática del quiz publicado hacia `mod.assessments` es un paso posterior — hoy el flujo emite el evento y el formador crea el quiz.

## Dependencias

Dura: `mod.courses` (resolver el curso y validar que la lección le pertenece).

## Modelo de datos

`mod_ai_content_draft` — el borrador: tipo, estado (`DRAFT`/`PUBLISHED`/`REJECTED`), contenido JSON (forma distinta por tipo) y telemetría IA.

## API

Prefijo `/modules/ai-content` (formador+): `generate`, listado/detalle de `drafts` y transiciones. Detalle en [Referencia → Pagos, aula e IA](../../api/referencia/pagos-directo-ia.md#ia).

## Eventos

**Emite**: `ai-content.draft.generated`, `ai-content.draft.published`, `ai-content.draft.rejected`. No consume.

## Configuración

Proveedor IA por tenant en Administración → Proveedores de IA; sin variables propias.
