# mod.ai-grader — AI grading with rubrics

<span class="didacta-chip didacta-chip--community">Community</span> · **AI** category (can be disabled)

## What it does

AI-assisted grading for the open-ended questions in [mod.assessments](assessments.md) (`SHORT_ANSWER`, `LONG_ANSWER`). The instructor defines a **rubric per question** (instructions + 1-8 weighted criteria); the AI reads the rubric and the student's answer and proposes a **score with a per-criterion justification** plus overall feedback. The score is **never published on its own**: the instructor reviews and applies (or discards) the suggestion. The module records AI–human agreement metrics.

## How it works

- The criteria weights must add up to the question's points (validated when the rubric is saved).
- There is **one suggestion per answer**: regenerating with `force` replaces the previous one; without `force`, the cached one is served without calling the model again.
- "Apply" on a suggestion only marks it as applied for auditing — the real score is set through the manual grading in assessments (`POST attempts/:id/grade`), keeping a single grading path.
- Suggestions are persisted so they can be reviewed later without spending tokens.

## Dependencies

Optional: `mod.assessments` (the source of attempts, answers and questions).

## Data model

`mod_ai_grader_rubric` (one per question: instructions + weighted criteria as JSON) · `mod_ai_grader_suggestion` (proposed score, per-criterion breakdown, provider/model, application traces).

## API

Prefix `/modules/ai-grader` (instructor and above): rubric management and the suggestion cycle. Details in [Reference → Payments, classroom and AI](../../api/referencia/pagos-directo-ia.md#ai).

## Events

**Emits**: `ai-grader.suggestion.generated`, `ai-grader.suggestion.failed`. It consumes none.

## Configuration

The AI provider under Administration → AI providers; no variables of its own.
