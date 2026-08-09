# mod.assessments — Quizzes and exams

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Assessments with auto-grading and manual grading. Six question types: `SINGLE_CHOICE`, `MULTIPLE_CHOICE` and `TRUE_FALSE` (graded automatically by a pure, testable scoring engine), `FILL_IN_BLANK` (accepted answers) and `SHORT_ANSWER`/`LONG_ANSWER` (which go to **manual grading by the instructor**, with a pending queue). Each quiz configures a pass threshold (60% by default), a maximum number of attempts, a time limit, question shuffling and whether feedback is shown on completion.

## How it works

- The quiz starts as `DRAFT`, questions are added and then it is published (at least one is required).
- The student sees a **solution-free view** (`preview`), starts an attempt, answers and submits; the engine grades the automatic part and, if there are open-ended questions, the attempt stays in `PENDING_REVIEW` until the instructor grades it from their panel.
- An attempt that has run out of time returns `ATTEMPT_EXPIRED` **410**.
- The instructor's view includes `isCorrect` on the options; the student's view never does.
- With [mod.ai-grader](ai-grader.md) enabled, the instructor gets per-criterion score suggestions with justifications for the open-ended questions.

## Dependencies

- Hard: `mod.courses`. Optional: `mod.learning` (auto-completes the QUIZ lesson on passing).

## Data model

| Table | What it stores |
| --- | --- |
| `mod_assessments_quiz` | The quiz: threshold, attempts, time, status. |
| `mod_assessments_question` | Statement, type, points, accepted answers. |
| `mod_assessments_option` | Options for choice questions. |
| `mod_assessments_attempt` | A student's attempt and its score. |
| `mod_assessments_answer` | The answer per question (option or text). |

## API

Prefix `/modules/assessments`, two surfaces: authoring/grading (instructor and above) and student. Details in [Reference → Learning](../../api/referencia/aprendizaje.md#assessments-modulesassessments).

## Events

**Emits**: `assessments.quiz.published`, `assessments.attempt.started/submitted/pending_review/graded/passed/failed`. It consumes none.

## Configuration

No per-tenant settings and no variables of its own: everything is configured per quiz.
