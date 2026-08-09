# mod.learning — Player, enrollment and progress

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

It manages the student's whole lifecycle inside a course: **enrollment** (direct, by an admin, by code or by invitation link), **per-lesson progress** (seconds watched, resume position) and **completion** with a configurable threshold (75% by default). On top of that:

- **Drip content**: release schedules based on intervals relative to the student's start date, segmentable by payment tier or by access group, with an optional email notice when a lesson unlocks.
- **Invitations** with a short code or a URL token, with a usage limit, an expiry date and revocation.
- **Lesson comments** with a moderation queue (they start as `PENDING` until the instructor approves them).
- **Learning paths** (course itineraries, linear or flexible) and per-course weighted **skills**.
- **SCORM 1.2/2004**: package upload, `imsmanifest.xml` parsing and persistence of the `cmi.*` state per attempt.

## How it works

- You can only enroll in `PUBLISHED` courses (`COURSE_NOT_PUBLISHED` 422); a duplicate active enrollment returns `ALREADY_ENROLLED` 409.
- When several drip schedules apply to a student, **the most permissive one wins** (the earliest unlock date). With no applicable schedule, everything is available.
- Trial content (membership trial) is gated with `TRIAL_CONTENT_LOCKED` 403, which the frontend turns into a payment CTA.
- On crossing the threshold it emits `learning.course.completed` **exactly once** — that is what triggers certificate issuing.

## Dependencies

- Hard: `mod.courses`.
- Optional: `mod.payment-connections` (tiers for drip), `mod.access-groups` (groups for drip), `mod.subscriptions` (trial status + lesson limit).

## Data model

`mod_learning_enrollment` (one enrollment per user and course) · `mod_learning_progress` · `mod_learning_invitation` · `mod_learning_drip_schedule` · `mod_learning_lesson_unlock_sub` · `mod_learning_scorm_package` · `mod_learning_scorm_attempt` · `mod_learning_lesson_comment` · `mod_learning_path` + `_path_course` + `_path_enrollment` · `mod_learning_competency` + `_course_competency`.

## API

Prefix `/modules/learning` (+ `/modules/learning/paths`). Full details in [Reference → Learning](../../api/referencia/aprendizaje.md#enrollments-progress-and-drip-moduleslearning).

## Events

- **Emits**: `learning.enrollment.created`, `learning.enrollment.cancelled`, `learning.progress.updated`, `learning.course.completed`, `learning.invitation.created`.
- **Consumes**: `courses.course.archived`.

## Configuration

No variables of its own: the completion threshold is per enrollment and drip is configured per course from the API/panel. The unlock notifier cron is `LESSON_UNLOCK_CRON`.
