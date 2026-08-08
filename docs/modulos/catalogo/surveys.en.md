# mod.surveys — Surveys and NPS

<span class="didacta-chip didacta-chip--community">Community</span> · **Engagement** category (can be disabled)

## What it does

**Anonymous** community surveys. The main use case: at the end of every live class (`zoom.session.ended`) the post-class survey is created automatically (NPS + rating), the registered attendees are notified and the admin sees the **aggregate results** (NPS, averages, free-text answers). It includes a reminder worker for anyone who has not answered after 24 h.

## How it works

- Anonymity is **structural**: the response stores no `userId`, only a `respondentHash` = HMAC(survey:user) with a server-side secret — enough to deduplicate (one response per person) without being able to identify the author.
- One survey per Zoom session and one reminder per survey (guaranteed with unique constraints in the database).
- The admin can create a class's survey without waiting for the webhook, close surveys (they stop accepting responses with a 409) and force the reminder sweep.
- The model also covers course-completion and general tenant surveys (`POST_COURSE`, `GENERAL`), which have no automatic trigger yet.

## Dependencies

Optional: `mod.zoom-live` (reading the registered attendees in order to notify them when the post-class survey is created).

## Data model

`mod_surveys_survey` (type, OPEN/CLOSED status, reminder flag) · `_question` (typed and positioned) · `_response` (anonymous, `respondentHash` only) · `_answer` (numeric or text value).

## API

Prefix `/modules/surveys` (student + admin). Details in [Reference → Community and people](../../api/referencia/comunidad.md#surveys-modulessurveys-anonymous-responses).

## Events

- **Emits**: `surveys.survey.created`, `surveys.response.submitted`.
- **Consumes**: `zoom.session.ended` (creates the post-class survey).

## Configuration

No per-tenant settings. Workers: `SURVEYS_REMINDER_CRON` (every 15 min), `SURVEYS_REMINDER_DELAY_HOURS` (24), `SURVEYS_HASH_SECRET` (falls back to `AUTH_SECRET`).
