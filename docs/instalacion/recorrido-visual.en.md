# Visual walkthrough: getting started

This guide walks through a Didacta installation **from scratch**, exactly as any new self-hoster would see it: the release image is built, `docker-compose.alpha.yml` is brought up with no pre-existing data, and the real path of an operator is completed — setup wizard, branding, first course, enrolling students and a certificate — with an annotated screenshot at every step.

!!! note "Where these screenshots come from"
    They were taken against an isolated, ephemeral instance with no seeds, started from a published release tag. The organization, course and people names are examples (`Academia Demo`, `Admin Demo`, `Alumna Demo`) — replace them with your own.

## 1 · First start and setup wizard

The first time you connect, Didacta redirects every route to `/setup`. The welcome screen summarises what you are about to configure in under a minute.

![Setup wizard welcome screen](../assets/recorrido-visual/en/01-setup-intro.png)

The second step asks for the organization name and the public domain (the host you are connecting from, editable).

![Organization step: name and public domain](../assets/recorrido-visual/en/02-setup-organizacion.png)

After choosing the active modules (core ones come pre-selected), the wizard asks for the first administrator's details — an account with the `super_admin` role.

![Administrator account step: name, email and password](../assets/recorrido-visual/en/03-setup-admin.png)

When it finishes, the organization and the administrator already exist and the session is active: there is no need to sign in again.

![Final wizard screen: platform created](../assets/recorrido-visual/en/04-setup-listo.png)

→ Full detail of what the wizard creates in [Setup wizard](setup-wizard.md).

## 2 · Complete your profile

The first time any account signs in (including the administrator's), Didacta asks for a minimal profile — a mandatory photo, name, language and notification preferences — before letting you into the rest of the platform.

![Onboarding: uploading a profile photo](../assets/recorrido-visual/en/05-onboarding-foto.png)

## 3 · Sign in

The sign-in screen (`/signin`) already carries the tenant's branding: name, real figures (active students, published courses) and the copy you configure under **Administration → Branding**.

![Sign-in screen with the tenant's branding](../assets/recorrido-visual/en/06-signin.png)

## 4 · Customise your branding

Under **Administration → Branding** (`/admin/branding`) you upload the logo and pick the brand color (`brandHue`). Both are applied instantly to the sign-in screen and the public catalog.

![Admin → Branding: logo uploaded and color chosen, before saving](../assets/recorrido-visual/en/07-branding.png)

## 5 · Create your first course

From **Instructor → Courses → New course** you set the title, description and category. The *slug* (public URL) is generated from the title. The course starts as a draft.

![Creating a new course, in draft](../assets/recorrido-visual/en/08-curso-nuevo.png)

Inside the course you add sections and, within each section, lessons — here, a text lesson.

![Course builder: adding a text lesson](../assets/recorrido-visual/en/09-curso-leccion-alta.png)

Publishing requires four things (title, description, at least one section and one lesson); the panel tells you what is missing at all times.

![Publishing checklist complete](../assets/recorrido-visual/en/10-curso-listo-publicar.png)

Once everything is in place, the course moves to **Published**.

![Course published](../assets/recorrido-visual/en/11-curso-publicado.png)

## 6 · Invite your students

Without public self-enrollment, the normal way for someone to get into a course is an **invitation code**, generated from the course itself.

![Invitation code generated for the course](../assets/recorrido-visual/en/12-curso-invitacion-generada.png)

!!! tip "The public catalog (`/catalogo`) is the sales catalog"
    `/catalogo` is the public listing for courses **with a price** (mod.billing) — designed for a visitor who pays by card without signing up first. A published course with no price configured, like the one in this walkthrough, does not appear there: it is still perfectly reachable by invitation or through the internal catalog once students are signed in. See [Three ways to fill your academy](../primeros-pasos/index.md#three-ways-to-fill-your-academy-with-students).

![Empty public sales catalog: this course has no price configured](../assets/recorrido-visual/en/13-catalogo-publico.png)

To add someone directly, **Administration → Users & roles → Invite person** creates the account already active and sends them an email to set their password.

![Adding a student through an individual invitation](../assets/recorrido-visual/en/14-admin-invitar-alumna.png)

## 7 · The student signs in and enrolls

From the link in the email, the student sets her password.

![The student sets her password from the email link](../assets/recorrido-visual/en/15-alumna-definir-password.png)

Once signed in, `/cursos` is the app's **internal** catalog: it lists every published course in the organization, priced or not (unlike the `/catalogo` sales listing from the previous step).

![Internal catalog at /cursos, signed in](../assets/recorrido-visual/en/16-alumna-catalogo.png)

On the course page, she redeems the invitation code the organization gave her.

![Redeeming the invitation code on the course page](../assets/recorrido-visual/en/17-alumna-canjear-codigo.png)

Redeeming it enrolls her instantly and shows her progress dashboard.

![Student enrolled: progress dashboard](../assets/recorrido-visual/en/18-alumna-matriculada.png)

## 8 · Complete the lesson and download the certificate

The lesson player shows the content and, for types such as text or PDF, a button to mark it as completed manually.

![Text lesson content](../assets/recorrido-visual/en/19-alumna-leccion.png)

Once the course completion threshold is reached, Didacta issues the certificate automatically and the hero button changes to "Download certificate".

![Course completed: certificate ready to download](../assets/recorrido-visual/en/20-alumna-curso-completado.png)

All of a student's certificates are also kept at `/mis-certificados`, with a verifiable sequential number.

![/mis-certificados with the issued certificate](../assets/recorrido-visual/en/21-alumna-mis-certificados.png)

## 9 · A look at Administration → License

Community works in full without a license. **Administration → License** (`/admin/licencia`) shows the current status and lets you activate an Enterprise license by pasting the key — verification is local, with no calls to an external server. If the operator pinned `DIDACTA_LICENSE_KEY` through the environment, the screen becomes read-only; see [License](../enterprise/licencia.md).

![Admin → License: Community status, no active Enterprise license](../assets/recorrido-visual/en/22-admin-licencia.png)

## What these screenshots do not show

- The own-enrollments endpoint (`GET /me/enrollments`) can take up to ~1 second to reflect a freshly created enrollment. If you have just redeemed a code and do not see the course immediately in some listing, reload.

## Next step

- [Visual walkthrough: notifications and sales](recorrido-visual-ventas.md) — configure real SMTP and get the instance ready to sell courses and memberships.
- [Configuration](../configuracion/index.md) — real SMTP, S3 storage, AI.
- [Managing modules](../modulos/gestion.md) — enable or disable optional modules.
- [Backups](copias-de-seguridad.md) — before there is any real data.
