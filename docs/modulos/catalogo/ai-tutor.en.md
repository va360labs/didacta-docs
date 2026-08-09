# mod.ai-tutor — Conversational tutor with RAG

<span class="didacta-chip didacta-chip--community">Community</span> · **AI** category (can be disabled)

## What it does

A **per-course** AI tutor with RAG: it indexes the content when the course is published, generates embeddings (provider configurable per tenant through the [AI Gateway](../../configuracion/ia.md)) and stores them in pgvector; the chat lives on the course page and **cites the specific lessons** each answer relies on.

On top of that there is a **quality loop** under Administration → AI Tutor with three tabs: review of question→answer pairs (with an "Unsupported only" filter to isolate answers with no citations), human-written **validated knowledge**, and a monthly report grouping questions by topic and ordering them by volume.

## How it works

- It indexes on receiving `courses.course.published` and **reindexes per lesson** when lessons are created/edited/deleted — there is no need to reindex the whole course when you add a class.
- **Human corrections live in their own table**, not as chunks: they are retrieved by similarity, injected into the prompt **above** the RAG context, and take effect immediately without reindexing (and they survive reindexes, which delete and regenerate the chunks).
- A quota of **10 questions per day** per student (it does not apply to staff, who can also try any course's tutor without enrolling).
- Quota exceeded → 429; AI provider not configured → 424.

## Dependencies

Optional: `mod.courses`, `mod.learning`.

## Data model

`mod_ai_tutor_conversation` · `_message` (with the question's embedding and its review status) · `_chunk` (indexed chunks, 1536-dimension vector) · `_correction` (validated knowledge, 1536-dimension vector) · `_token_usage` (consumption, for the quota).

## API

`POST /modules/ai-tutor/courses/:courseId/ask` (student) + the admin surface (`/admin/ai-tutor/*`: indexing, review, corrections, report). Details in [Reference → Payments, classroom and AI](../../api/referencia/pagos-directo-ia.md#ai).

## Events

- **Emits**: `ai-tutor.course.indexed/index-failed`, `ai-tutor.chat.message-sent/message-received`, `ai-tutor.answer.reviewed/corrected`.
- **Consumes**: `courses.course.published/unpublished`, `courses.lesson.created/updated/deleted`.

## Configuration

Provider and key under Administration → AI providers (requires `AI_CONFIG_ENCRYPTION_KEY`). The RAG parameters (top-K 5, a 3000-token history budget, the corrections threshold, the daily quota of 10) are product constants.
