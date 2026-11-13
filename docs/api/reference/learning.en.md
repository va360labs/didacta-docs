# Reference — Learning

Endpoints for courses, enrollments and progress, learning paths, assessments, certificates, access groups, groups, events and Fundae. Every route hangs off `/api/v1`.

**Auth legend**: `Bearer` = any authenticated user of the tenant · `instructor+` = instructor, tenant_admin or super_admin · `admin` = tenant_admin or super_admin · `Public` = no session.

## Courses — `/modules/courses`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/courses` | Bearer | Lists the tenant's courses; query `status`, `q`, `category`. |
| POST | `/modules/courses` | instructor+ | Creates a course in DRAFT. |
| GET | `/modules/courses/categories` | Bearer | Categories used by published courses. |
| GET | `/modules/courses/managed-categories` | Bearer | The tenant's curated categories (color, icon). |
| POST | `/modules/courses/managed-categories` | admin | Creates a curated category. |
| PUT | `/modules/courses/managed-categories/:id` | admin | Updates a curated category. |
| DELETE | `/modules/courses/managed-categories/:id` | admin | Deletes a curated category. |
| GET | `/modules/courses/:id` | Bearer | Detail with modules and lessons (see the gating below). |
| PUT | `/modules/courses/:id` | instructor+ | Updates the course metadata. |
| POST | `/modules/courses/:id/modules` | instructor+ | Adds a module to the course. |
| POST | `/modules/courses/modules/:moduleId/lessons` | instructor+ | Adds a lesson to the module. |
| PUT | `/modules/courses/lessons/:lessonId` | instructor+ | Updates a lesson's content. |
| POST | `/modules/courses/:id/publish` | instructor+ | Publishes it (running the `courses.publish.validate` hook). |
| POST | `/modules/courses/:id/archive` · `/:id/unarchive` | instructor+ | Archives it / returns it to DRAFT. |
| POST | `/modules/courses/lessons/:lessonId/move` | instructor+ | Moves the lesson one position up/down. |
| POST | `/modules/courses/lessons/:lessonId/move-to-module` | instructor+ | Moves the lesson to another module. |
| POST | `/modules/courses/modules/:moduleId/reorder-lessons` | instructor+ | Reorders lessons in bulk (drag & drop). |
| POST | `/modules/courses/:id/reorder-modules` | instructor+ | Reorders the course's modules in bulk. |
| DELETE | `/modules/courses/modules/:moduleId` | instructor+ | Soft deletes the module (with a logical cascade over its lessons). |
| DELETE | `/modules/courses/lessons/:lessonId` | instructor+ | Soft deletes the lesson (preserving historical progress). |

**Key bodies** — creating a course: `slug` (kebab-case), `title`, `description?`, `thumbnailUrl?`, `language` (default `es-ES`), `estimatedMinutes?`, `category?`. `slug` and `language` are immutable. Creating a lesson: `type` (`VIDEO|HTML|PDF|TEXT|QUIZ|SCORM`), `title`, `content` (a free-form object), `durationMinutes?`, `publishAt?` (a future date = locked).

**Read gating on `GET /:id`**: instructor+ receives the full course; a student gets a 404 if it is not `PUBLISHED`, the structure with `content: null` if they are not enrolled, and `content: null` only on lessons that have not been released yet when drip is in play.

**Errors**: `COURSE_NOT_FOUND` 404 · `COURSE_SLUG_EXISTS` 409 · `COURSE_ALREADY_PUBLISHED` 409 · `COURSE_NO_LESSONS` 422 · `COURSE_PUBLISH_VALIDATION_FAILED` 422 (with a `reasons` array).

## Enrollments, progress and drip — `/modules/learning`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/learning/me/enrollments` | Bearer | My enrollments. |
| GET | `/modules/learning/me/stats` | Bearer | My statistics (courses completed, time watched). |
| GET | `/modules/learning/me/enrollments/:id/progress` | Bearer | My per-lesson progress for one enrollment. |
| POST | `/modules/learning/enrollments/me` | Bearer | Self-enrollment in a course. |
| POST | `/modules/learning/enrollments/by-code` · `by-link` | Bearer | Self-enrollment with an invitation code / token. |
| POST | `/modules/learning/enrollments` | instructor+ | Enrolls another user (origin ADMIN). |
| DELETE | `/modules/learning/enrollments/:id` | Bearer | Cancels my enrollment. |
| DELETE | `/modules/learning/enrollments/:id/by-admin` | instructor+ | Removes a student's enrollment. |
| POST | `/modules/learning/progress` | Bearer | Reports progress: `{ enrollmentId, lessonId, watchedSeconds, resumePositionSec?, completed? }`. |
| GET | `/modules/learning/courses/:courseId/enrollments` | instructor+ | Students enrolled in the course. |
| GET | `/modules/learning/courses/:courseId/enrollments/:id/progress` | instructor+ | A student's detailed progress. |
| GET | `/modules/learning/invitations` | instructor+ | Active invitations for a course (`?courseId=`). |
| POST | `/modules/learning/invitations` | instructor+ | Creates an invitation (`courseId`, `maxUses?`, `expiresAt?`) → code + token. |
| DELETE | `/modules/learning/invitations/:id` | instructor+ | Revokes an invitation. |
| GET | `/modules/learning/courses/:courseId/drip` | instructor+ | The course's drip schedules. |
| POST | `/modules/learning/courses/:courseId/drip` | instructor+ | Creates a schedule: `audienceKind` (`TIER\|GROUP`), `audienceRef`, `unit` (`LESSON\|MODULE`), `intervalDays` (≥1), `startOffsetDays?`. |
| PUT | `/modules/learning/drip/:id` · DELETE | instructor+ | Edits / deletes a drip schedule. |
| GET | `/modules/learning/courses/:courseId/availability` | Bearer | Unlock dates for the lessons, for the current student. |
| GET/POST/DELETE | `/modules/learning/lessons/:lessonId/unlock-subscription` | Bearer | Reads / subscribes to / unsubscribes from the unlock email notice. |
| GET | `/modules/learning/lessons/:lessonId/comments` | Bearer | Comments (everyone's APPROVED ones + your own; instructor+ also sees PENDING). |
| POST | `/modules/learning/lessons/:lessonId/comments` | Bearer | Creates a comment (it starts as PENDING until moderated). |
| GET | `/modules/learning/courses/:courseId/comments/pending` | instructor+ | The course's moderation queue. |
| POST | `/modules/learning/comments/:id/approve` · `reject` | instructor+ | Moderates a comment (`reject` accepts a `reason?`). |
| DELETE | `/modules/learning/comments/:id` | Bearer (author) | Deletes your own comment. |
| GET | `/modules/learning/me/competencies` · `/modules/learning/competencies` | Bearer | My skills map / the tenant's catalog. |
| POST · DELETE | `/modules/learning/competencies[/:id]` | instructor+ | Creates / deletes a skill. |
| GET · PUT | `/modules/learning/courses/:courseId/competencies` | Bearer · instructor+ | The course's skills / replaces the whole set (`items: [{competencyId, weight?}]`). |
| POST | `/modules/learning/lessons/:lessonId/scorm` | instructor+ | Uploads a SCORM 1.2/2004 package as base64 (max. ~100 MiB of binary). |
| GET | `/modules/learning/lessons/:lessonId/scorm` | Bearer | Metadata + a signed entry URL for the iframe (an active enrollment is required, except for editors). |
| POST | `/modules/learning/lessons/:lessonId/scorm/attempt` · `commit` | Bearer | Starts/resumes the SCORM attempt · persists the `cmi` state (on completion it bridges to progress). |
| POST | `/modules/learning/lesson-unlock/run-now` | super_admin | Forces one cycle of the unlock notifier (QA). |

**Errors**: `ALREADY_ENROLLED` 409 · `ENROLLMENT_NOT_FOUND` 404 · `INVITATION_INVALID` 400 · `COURSE_NOT_PUBLISHED` 422 · `LESSON_LOCKED` 403 · `TRIAL_CONTENT_LOCKED` 403 (trial content, unlocked by paying) · `SCORM_*` 400/404/422.

## Learning paths — `/modules/learning/paths`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/learning/paths` | Bearer | Published paths with my progress. |
| GET | `/modules/learning/me/paths` | Bearer | My paths (active + completed). |
| GET | `/modules/learning/paths/formador` | instructor+ | Every path in any state (the instructor's panel). |
| GET | `/modules/learning/paths/:slug` | Bearer | Detail of a published path with its courses. |
| POST | `/modules/learning/paths` | instructor+ | Creates a path: `title`, `description?`, `sequenceType?` (`LINEAR\|FLEXIBLE`). |
| PATCH | `/modules/learning/paths/:id` | instructor+ | Updates it (including `courses: [{courseId, position}]` — this replaces the set). |
| POST | `/modules/learning/paths/:id/publish` · `archive` · `restore` | instructor+ | Publishes/unpublishes · archives · restores to DRAFT. |
| POST · DELETE | `/modules/learning/paths/:id/enroll` | Bearer | Enrollment in the path (and its courses) · cancellation. |

**Errors**: path not found / not published 404 · already enrolled 409 · path with no courses 400.

## Assessments — `/modules/assessments`

**Management (instructor+):**

| Method | Route | What it does |
|---|---|---|
| POST | `/modules/assessments/quizzes` | Creates a quiz in DRAFT: `title`, `lessonId?`, `passThreshold?` (0-100), `maxAttempts?`, `timeLimitMinutes?`, `shuffleQuestions?`, `showFeedback?`. |
| GET · PUT | `/modules/assessments/quizzes/:id` | Detail for the instructor (including `isCorrect`) · update. |
| POST | `/modules/assessments/quizzes/:id/questions` | Adds a question: `type` (`SINGLE_CHOICE\|MULTIPLE_CHOICE\|TRUE_FALSE\|FILL_IN_BLANK\|SHORT_ANSWER\|LONG_ANSWER`), `prompt`, `options?`, `acceptedAnswers?`, `points?`. |
| DELETE | `/modules/assessments/quizzes/:id/questions/:questionId` | Soft deletes the question. |
| POST | `/modules/assessments/quizzes/:id/publish` | Publishes it (at least 1 question required). |
| GET | `/modules/assessments/attempts/pending` | Attempts in `PENDING_REVIEW` (manual grading of open-ended answers). |
| GET | `/modules/assessments/attempts/:id/full` | The complete attempt, for the grader. |
| POST | `/modules/assessments/attempts/:id/grade` | Grades manually: `{ grades: [{questionId, scoreEarned, feedback?}] }`; emits `assessments.attempt.passed/failed`. |

**Student (Bearer):**

| Method | Route | What it does |
|---|---|---|
| GET | `/modules/assessments/quizzes/:id/preview` | The quiz view **without** solutions. |
| POST | `/modules/assessments/attempts` | Starts an attempt: `{ quizId, enrollmentId?, lessonId? }`. |
| POST | `/modules/assessments/attempts/:id/submit` | Submits answers `{ answers: [{questionId, selectedOptionIds?, textAnswer?}] }`; auto-grading + events. |
| GET | `/modules/assessments/attempts/:id` | Detail of one of your own attempts. |
| GET | `/modules/assessments/attempts?quizId=` | My attempts at a quiz. |

**Errors**: `QUIZ_NOT_PUBLISHED` / `QUIZ_HAS_NO_QUESTIONS` 422 · `ATTEMPT_ALREADY_SUBMITTED` / `MAX_ATTEMPTS_REACHED` 409 · `ATTEMPT_EXPIRED` **410** · not found 404.

## Certificates — `/modules/certificates`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/certificates/me` | Bearer | My issued certificates. |
| GET | `/modules/certificates/:id` · `/:id/download` | Holder or instructor+ | Detail · PDF download (regenerated from the immutable snapshot). A user can only reach their own certificates; staff can reach anyone's. Someone else's id returns 404. |
| GET | `/modules/certificates/verify/:id` | **Public** | Public verification: `{ number, studentName, courseTitle, issuedAt, valid }`. It never exposes an email address or internal data. |
| GET · POST | `/modules/certificates/templates` | instructor+ | Lists · creates a template: `name`, `body`, `primaryColor?`, `logoUrl?`, `signerName?`, `signerTitle?`, `isDefault?`. |
| GET · PATCH · DELETE | `/modules/certificates/templates/:id` | instructor+ | Detail · edit · delete (409 if it is the default or is in use). |
| POST | `/modules/certificates/templates/preview` | instructor+ | A preview PDF with dummy data, persisting nothing. |
| POST | `/modules/certificates/templates/:id/set-default` | instructor+ | Marks it as the tenant's default template. |

## Access groups — `/modules/access-groups` (all admin)

| Method | Route | What it does |
|---|---|---|
| GET | `/modules/access-groups` | Paginated list (`page`, `limit`). |
| GET | `/modules/access-groups/catalog/courses` · `catalog/users` | Pickers: published courses · candidate users (`?q=`). |
| GET | `/modules/access-groups/:id` | Detail with courses and members. |
| POST | `/modules/access-groups` | Creates one: `name`, `slug?`, `kind` (`ALL_COURSES\|COURSE\|MULTI_COURSE`), `courseIds?`, `autoGrantNewCourses?`. |
| PATCH | `/modules/access-groups/:id` | Edits it (`name`, `description`, `autoGrantNewCourses`, `isDefaultForApproval`, `linkedTierName` — links a payment tier). |
| PUT | `/modules/access-groups/:id/courses` | Replaces the whole set of courses. |
| POST | `/modules/access-groups/:id/members` | Assigns members: `{ userIds: [] }` (max. 500). |
| DELETE | `/modules/access-groups/:id/members/:userId` | Revokes a member. |
| DELETE | `/modules/access-groups/:id` | Deletes the group and revokes its memberships. |

## Community groups and events

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/groups` · `/me` · `/:id` | Bearer | Paginated list · my groups · detail with members. |
| POST | `/modules/groups` | instructor+ | Creates a group (`name`, `slug`, `description?`); the creator becomes the owner. |
| POST · DELETE | `/modules/groups/:id/join` · `/:id/leave` | Bearer | Join · leave (idempotent). |
| GET | `/modules/events` · `/:id` | Bearer | Events by date range (`from`, `to`, `limit`, `order`) · detail with `registeredCount`, `isFull`, `isRegistered`. |
| POST | `/modules/events` | instructor+ | Creates an event: `title`, `startAt`, `endAt`, `location?`, `capacity?`. |
| POST | `/modules/events/:id/register` · `unregister` | Bearer | Registration (if it is full: `{ registered: false, reason: 'full' }`) · cancellation. |

## Fundae — `/modules/fundae` (all admin; the instructor role has no access)

| Method | Route | What it does |
|---|---|---|
| GET · POST | `/modules/fundae/actions` | List (filters `courseId`, `status`) · creates a training action: `codigoAccion` (≤25), `nombre`, `modalidad` (`PRESENCIAL\|TELEFORMACION\|MIXTA`), `horasFormacion`, `fechaInicio`/`fechaFin` (`YYYY-MM-DD`), `courseId?`, `lugar?`, `cifCentro?`. |
| GET · PUT · DELETE | `/modules/fundae/actions/:id` | Detail · update (+ `status`) · archive (soft). |
| GET | `/modules/fundae/actions/:id/participants` · `/count` | Participants with email, national ID, progress and result · the count. |
| GET | `/modules/fundae/actions/:id/export.xml` | The action's Fundae XML (download). |
| GET | `/modules/fundae/actions/:id/participants/:userId/evidence.pdf` | A signed evidence PDF for one participant. |
| GET | `/modules/fundae/actions/:id/export.zip` | The submission ZIP: the XML + one evidence PDF per participant. |
| GET · POST | `/modules/fundae/actions/:id/blocks` | Training blocks · creation (`ordinal?`, `title`, `hours`, `modalidad`, `contenidos?`). |
| PUT · DELETE | `/modules/fundae/actions/:id/blocks/:blockId` | Editing · deleting a block. |

The block hours cannot add up to more than the action's `horasFormacion` (Fundae checks this when the XML is uploaded). The signatory on the evidence is the administrator who triggers the download.
