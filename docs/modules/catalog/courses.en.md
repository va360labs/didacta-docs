# mod.courses — Courses and catalog

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

This is the heart of the catalog: courses with a `slug`, title and status (`DRAFT` → `PUBLISHED` → `ARCHIVED`), **modules** as an ordered grouping inside a course, and typed **lessons** (`VIDEO`, `HTML`, `PDF`, `TEXT`, `QUIZ`, `SCORM`) with flexible JSON content. It includes admin-curated categories (name, color, icon) and advanced editing operations: bulk reordering of modules and lessons (drag & drop) and moving lessons between modules.

## How it works

- **Publishing goes through an open hook**: before publishing, `mod.courses` runs `courses.publish.validate` and any registered module can add blocking reasons (for example, `mod.fundae` requires objectives and a duration). If there are any reasons, publishing fails with a 422 and the list of reasons.
- Deletions are **soft deletes** with a logical cascade: removing a module or a lesson preserves students' historical progress.
- It is the piece almost every other module hangs off (learning, access-groups, billing, assessments, ai-content, ai-tutor…).

## Configuration

- **Activation**: a `core`-category module, always active in every tenant. It does not appear in the "Modules" tab at `/admin/settings?tab=modules` (that tab only lists deactivatable modules) and the API refuses to disable it; if a row is ever left with `enabled=false` in the database, server startup re-enables it automatically.
- **Per-tenant settings**: none. All configuration is per course, from the instructor's builder.
- **Curated categories**: the administrator manages them at `/admin/courses/categories` ("Course categories" screen): name, "Color" and "Icon". The name is an exact-text match with the category written on the course; the catalog and the course card render the chip with that style.
- **Environment variables**: none belongs to the module itself. Content-related ones: `LESSON_IFRAME_ALLOWED_HOSTS` (extra `<iframe>` hosts in HTML lessons) and `BUNNY_STREAM_*` (video upload, Didacta Cloud only; see "Lesson video"). Cover images and SCORM packages are stored in the instance's file backend (`/admin/settings?tab=storage`, local disk or S3-compatible).
- **License**: the whole module is Community; no feature requires Enterprise capabilities.

!!! note "Screenshots in full mode"
    The screenshots on this page come from **full mode** (side menu), the mode a new installation starts in. In **simple mode** the same routes are still available from "More", except for the screens that mode replaces with a simplified version (the course editor, Settings, Sales, Students, Live, the community and Referrals); to use them as shown here, switch to full mode under **Settings → "Use the full mode"**.

## Step-by-step usage

### Creating and building a course (instructor)

1. At `/studio/courses` ("My courses"), click "Create new course". Fill in "Title" and "Slug" (auto-generated from the title, kebab-case: it will be the course's public URL) and, optionally, a description, "Category" and "Estimated duration (min)". Click "Create course": the course starts in `DRAFT`.

    ![The instructor's "My courses" list with draft, published and archived courses](../../assets/modules/courses/en/courses-1-my-courses.png)

2. In the builder (`/studio/courses/<id>`), click "Add section", type the title and confirm with "Create section". Sections group related lessons.
3. Inside each section, "Add lesson": pick the "Type" (Video, HTML, PDF, Text, Quiz or SCORM), type the title and click "Create".
4. Click "Edit" on the lesson to load its content depending on the type: "Video URL" (YouTube, Bunny Stream or a `.mp4`/`.webm`/`.m3u8` file) with clickable "Resources and chapters" (and, optionally, "Transcript (for the AI tutor)": paste the text or upload an `.srt`, `.vtt` or `.txt`; students do not see it, the [AI tutor](ai-tutor.md) reads it), "PDF URL", "HTML", "Text", a linked quiz (see [mod.assessments](assessments.md)) or "Upload SCORM package" (a `.zip` with `imsmanifest.xml` at the root). The "Publish date (optional)" field schedules the lesson: with a future date it shows in the list but stays locked until then.

    ![The course builder with sections, lessons and the publish card](../../assets/modules/courses/en/courses-2-builder.png)

5. "Edit" in the header adjusts the metadata: description, "Category", "Estimated duration (min)", "Featured image" (the "Optimize" button recompresses it to WebP), "Featured video (URL)" and "External purchase URL" if this course is sold on an external page.
6. Reorder sections and lessons by dragging (drag & drop); a lesson can also be moved to another section. Deletions are soft deletes: the item stops showing but students' historical progress is preserved.
7. The "Before publishing" card lists the requirements: "Course title", "Description", "At least one section" and "At least one lesson". Once everything is met it switches to "Ready to publish". Click "Publish"; if a module hooked into the validation blocks publishing (e.g. FUNDAE), the API responds 422 with the reasons.
8. "Archive" removes the course from the catalog without deleting anything; "Unarchive" returns it to `DRAFT` (it is not republished automatically: you review it and click "Publish" again).

### Lesson video: upload and protection

- **Uploading the video from the editor (Didacta Cloud only).** In a video lesson, besides pasting a link, the editor offers "Upload the video for this lesson": drag the file or pick it, with "Pause" and "Resume" if the connection drops. After the upload the video is prepared in a few minutes (you can keep editing) and you must click "Save changes" for students to see it. It only exists if the instance has `BUNNY_STREAM_API_KEY`; without it (the case of a self-hosted install) the editor only accepts links and the `/video/uploads` routes answer 404. Limits per video: 4 GB and 2 hours. The total space depends on whether you have sold yet: before the first sale, 5 h stored, up to 720p and 100 h watched per month; afterwards, 50 h (plus 10 h for every €30 of operator commission over the last 12 months), up to 1080p and no cap on hours watched. The editor's "Your video space" card shows what you have used.
- **Video protection** (`/admin/security`, "Video protection" card, `super_admin` and `tenant_admin` only). For the Bunny Stream videos you link by hand: with protection on, every playback checks that the student receives that lesson's content and hands them a signed link that expires after 30 minutes. To turn it on you save each library's "Token authentication key" (stored encrypted and never shown again), enable the token in Bunny and click "Check". Settings: "Protection enabled" (on by default), "Watermark with the student email" (on by default), "Different videos per hour" (30) and "Different videos per day" (100) per account; instructors and admins have no cap. Videos uploaded from the editor are already signed by the platform and need no key.
- **Sanitized HTML.** Lesson HTML and the course description are sanitized **on the server when saving**: `<script>`, `<style>` and `javascript:` links are discarded. In HTML lessons only `<iframe>`s from YouTube, Vimeo and Bunny Stream are allowed (plus any hosts the operator adds in `LESSON_IFRAME_ALLOWED_HOSTS`, comma-separated).
- **Who receives the content.** `GET /modules/courses/:id` returns the full syllabus to a student, but each lesson's content only if they have a live enrollment and the lesson is released (drip, publish date, trial period or level); instructors and admins always see it complete.

### Curating the catalog categories (admin)

1. At `/admin/courses/categories`, create each category with "Name", "Color" and "Icon" and click "Create category". The preview shows the resulting chip.
2. The builder offers this list to the instructor as the "Category" dropdown. If a category is deleted, courses that used it keep showing the name as plain text.

    ![The curated categories screen with color and icon](../../assets/modules/courses/en/courses-3-categories.png)

### What the student sees

- The catalog (`/courses`) lists only `PUBLISHED` courses, with search and filters by topic and language.
- The course page (`/courses/<slug>`) shows the hero (featured video > image > fallback), the description and the syllabus. An instructor or admin can open the page of an unpublished course in "Preview" mode, without enrolling.

    ![The student's course catalog with category chips](../../assets/modules/courses/en/courses-4-catalog.png)

## Dependencies

- Hard: none.
- Optional: `mod.certificates` (validates the certificate template assigned to the course).

## Data model

| Table | What it stores |
| --- | --- |
| `mod_courses_course` | The course: slug, title, status, category, certificate template. |
| `mod_courses_module` | An ordered grouping inside the course. |
| `mod_courses_lesson` | The lesson: type, JSONB content, duration, scheduled publication date. |
| `mod_courses_category` | Curated categories (name, color, icon). |

## API

Prefix `/modules/courses`: CRUD for courses/modules/lessons, managed categories, status transitions and the bulk ordering endpoint. Details in [Reference → Learning](../../api/reference/learning.md#courses-modulescourses).

## Events and hooks

- **Emits**: `courses.course.created/updated/published/archived/unarchived`, `courses.module.created/deleted`, `courses.lesson.created/updated/moved/moved-to-module/deleted`, `courses.lessons.reordered` and `courses.modules.reordered`.
- **Exposes the hook** `courses.publish.validate` (the only active hook in the product — the reference pattern).
