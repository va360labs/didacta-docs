# mod.ai-content — Learning content generator

<span class="didacta-chip didacta-chip--community">Community</span> · **AI** category (can be disabled)

## What it does

It generates **learning drafts** from a lesson's text: summaries (`SUMMARY`), flashcards (`FLASHCARDS`) and quizzes (`QUIZ`). The result is always persisted as a **draft**: the instructor reviews it, can edit the JSON, and then publishes or rejects it with a reason. **Human-in-the-loop by default** — nothing is published automatically.

## How it works

- The prompt is specific to each type and the call goes through the [AI Gateway](../../configuracion/ia.md) with the tenant's configuration.
- Every draft records generation telemetry: provider, model and input/output tokens.
- A lesson with no text returns `AI_CONTENT_LESSON_TEXT_EMPTY` 422; with no provider configured, a 503 with a clear message.
- Only a draft in `DRAFT` state can be edited/published/rejected (409 in any other state).
- The instructor waits for the generation (~5-30 s); there is no streaming yet. Automatic ingestion of a published quiz into `mod.assessments` is a later step — today the flow emits the event and the instructor creates the quiz.

## Dependencies

Hard: `mod.courses` (to resolve the course and validate that the lesson belongs to it).

## Data model

`mod_ai_content_draft` — the draft: type, state (`DRAFT`/`PUBLISHED`/`REJECTED`), JSON content (a different shape per type) and AI telemetry.

## API

Prefix `/modules/ai-content` (instructor and above): `generate`, `drafts` listing/detail and state transitions. Details in [Reference → Payments, classroom and AI](../../api/referencia/pagos-directo-ia.md#ai).

## Events

**Emits**: `ai-content.draft.generated`, `ai-content.draft.published`, `ai-content.draft.rejected`. It consumes none.

## Configuration

The AI provider per tenant under Administration → AI providers; no variables of its own.
