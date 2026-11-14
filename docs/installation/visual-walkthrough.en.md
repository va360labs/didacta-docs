# Visual walkthrough: getting started

This guide walks through a Didacta installation **from scratch**, exactly as any new self-hoster would see it: an instance with no pre-existing data and the real path of an operator — setup wizard, the academy's welcome wizard, first course, enrolling students and a certificate — with a screenshot at every step.

!!! note "Where these screenshots come from"
    They were taken against an isolated, ephemeral instance with no seeds, running version 0.1.0-beta.18. The academy, course and people are examples (`Morning Pour`, `Ivy Carter`, `Theo Park`, domain `morningpour.example.com`) — replace them with your own.

    A new installation starts in **full mode** (side menu with every option); the [last step](#10-simple-mode) shows how to switch to **simple mode**, the mode Didacta Cloud academies start in.

## 1 · First start and setup wizard

While the instance has no organization, Didacta redirects every route to `/setup`. The wizard asks for a **single-use token** that the instance prints to its logs on startup: open it with the full `/setup?token=…` link (`docker logs didacta-app | grep "Setup token"`; the `install.sh` installer already shows it to you when it finishes). The welcome screen summarises what you are about to configure in under a minute.

![Setup wizard welcome screen](../assets/visual-walkthrough/en/01-setup-welcome.png)

The second step asks for the organization name and the public domain (the host you are connecting from, editable).

![Organization step: name and public domain](../assets/visual-walkthrough/en/02-setup-organization.png)

After choosing the active modules (core ones come pre-selected), the wizard asks for the first administrator's details — an account with the `super_admin` role.

![Administrator account step: name, email and password](../assets/visual-walkthrough/en/03-setup-admin.png)

When it finishes, the organization and the administrator already exist and the session is active: there is no need to sign in again.

![Final wizard screen: platform created](../assets/visual-walkthrough/en/04-setup-done.png)

→ Full detail of what the wizard creates in [Setup wizard](setup-wizard.md).

## 2 · The academy's welcome wizard

When you click "Get started", Didacta opens the welcome wizard (`/welcome`): six steps — **Name**, **Logo & color**, **First course**, **Students**, **Payments** and **Email** — that make the academy presentable without leaving the screen. Only the first two are mandatory; the rest have "Not now".

In **Logo & color** you upload the logo, choose how it is shown ("Logo only" or "Logo and name") and the brand color, with a preview of how your students will see it. Everything can be changed later under **Administration → Branding** (`/admin/branding`).

![Welcome wizard, "Logo & color" step: logo uploaded, "Logo and name" and brand color](../assets/visual-walkthrough/en/05-welcome-brand.png)

In **First course** you type a title and the course is created as a draft; in **Students** you paste email addresses (one per line) and each person gets an invitation to set their password. **Payments** and **Email** embed the same Stripe and SMTP cards as Settings ([second part of the walkthrough](visual-walkthrough-sales.md)). The final summary shows the academy's address and what is still pending.

![Wizard summary: the academy's address and pending steps](../assets/visual-walkthrough/en/06-welcome-summary.png)

## 3 · Complete your profile

The first time any account signs in (including the administrator's), Didacta asks for a minimal profile — a mandatory photo, name, language and notification preferences — before letting you into the rest of the platform.

![Onboarding: uploading a profile photo](../assets/visual-walkthrough/en/07-onboarding-photo.png)

When you finish you land in the community, with the "Getting started" card that ticks off what is done and what is missing.

## 4 · The sign-in screen

The sign-in screen (`/signin`) already carries the academy's branding: logo, color, real figures (active students, included courses) and the headline you configure under **Administration → Branding**.

![Sign-in screen with the academy's branding](../assets/visual-walkthrough/en/08-signin.png)

## 5 · Build and publish your first course

The wizard left the course as a draft. Open it from **Instructor → My courses** (`/studio/courses`) → "Edit": the header's "Edit" button lets you fill in the description, category and cover. Inside the course you add sections and, within each section, lessons — here, a text lesson.

![Course builder: adding a text lesson](../assets/visual-walkthrough/en/09-course-new-lesson.png)

Publishing requires four things (title, description, at least one section and one lesson); the "Before publishing" card tells you what is missing at all times and turns into "Ready to publish" when everything is in place.

![The "Ready to publish" card with the four requirements met](../assets/visual-walkthrough/en/10-course-ready.png)

With "Publish", the course moves to **Published**.

![Course published](../assets/visual-walkthrough/en/11-course-published.png)

## 6 · Invite your students

Without public self-enrollment, the normal way for someone to get into a course without paying is an **invitation code**, generated from the course's own "Invitations" card.

![Invitation code generated for the course](../assets/visual-walkthrough/en/12-course-invitation-code.png)

!!! tip "The public catalog (`/catalog`) is the sales catalog"
    `/catalog` is the public listing for courses **on sale** — designed for a visitor who pays by card without signing up first. A published course with no price, like the one in this walkthrough, does not appear there: it is still reachable by invitation or through the internal catalog once students are signed in. See [Three ways to fill your academy](../getting-started/index.md#three-ways-to-fill-your-academy-with-students).

![Empty public sales catalog: this course has no price](../assets/visual-walkthrough/en/13-public-catalog.png)

To add someone directly (on top of the people you invited in the wizard), **Administration → Users & roles → Invite person** (`/admin/users/invite`) creates the account and sends them an email to set their password.

![Adding a student through an individual invitation](../assets/visual-walkthrough/en/14-admin-invite-student.png)

## 7 · The student signs in and enrolls

From the link in the email, the student sets his password; then he signs in with it and completes his profile (step 3).

![The student sets his password from the email link](../assets/visual-walkthrough/en/15-student-set-password.png)

Once signed in, `/courses` is the app's **internal** catalog: it lists every published course in the organization, priced or not (unlike the `/catalog` sales listing from the previous step).

![Internal catalog at /courses, signed in](../assets/visual-walkthrough/en/16-student-catalog.png)

On the course page, he redeems the invitation code the organization gave him.

![Redeeming the invitation code on the course page](../assets/visual-walkthrough/en/17-student-redeem-code.png)

Redeeming it enrolls him instantly: he sees his progress, the syllabus and the open lesson. For types such as text or PDF, the "Mark as completed" button records the lesson manually.

![Student enrolled: progress, syllabus and a text lesson](../assets/visual-walkthrough/en/18-student-lesson.png)

## 8 · Complete the course and download the certificate

Once the course completion threshold is reached, Didacta issues the certificate automatically and the header button changes to "Download certificate".

![Course completed: certificate ready to download](../assets/visual-walkthrough/en/19-student-course-completed.png)

All of a student's certificates are also kept at `/my-certificates`, with a verifiable sequential number, PDF download and "Add to LinkedIn".

![/my-certificates with the issued certificate](../assets/visual-walkthrough/en/20-student-my-certificates.png)

## 9 · A look at Administration → License

Community works in full without a license. **Administration → License** (`/admin/license`) shows the current status and lets you activate an Enterprise license by pasting the key — verification is local, with no calls to an external server. If the operator pinned `DIDACTA_LICENSE_KEY` through the environment, the screen becomes read-only; see [License](../enterprise/license.md).

![Admin → License: Community status, no active Enterprise license](../assets/visual-walkthrough/en/21-admin-license.png)

## 10 · Simple mode

Didacta has two classroom modes. **Full mode** — the one a new installation starts in and the one in every screenshot above — shows every option in a side menu. **Simple mode** keeps a few tabs at the top ("Start here" —"Home" after the first sale—, "Community", "Courses", "Live", "Students", "Sales", "Settings"), editors without advanced options and everything else under "More"; it is the mode Didacta Cloud academies start in. To switch, click **"Back to the simple mode"** under **Administration → Settings** (`/admin/settings`); nothing is deleted.

![Settings: the "Back to the simple mode" card](../assets/visual-walkthrough/en/22-admin-simple-mode-card.png)

In simple mode, whoever runs the academy lands on "Start here" (`/start`), with what is left before the first sale. To go back, **Settings → "Use the full mode"**.

![Simple mode: "Start here" with the tabs at the top](../assets/visual-walkthrough/en/23-simple-mode-start.png)

## What these screenshots do not show

- The own-enrollments endpoint (`GET /me/enrollments`) can take up to ~1 second to reflect a freshly created enrollment. If you have just redeemed a code and do not see the course immediately in some listing, reload.

## Next step

- [Visual walkthrough: notifications and sales](visual-walkthrough-sales.md) — configure real SMTP and get the instance ready to sell courses and memberships.
- [Configuration](../configuration/index.md) — real SMTP, S3 storage, AI.
- [Managing modules](../modules/managing.md) — enable or disable optional modules.
- [Backups](backups.md) — before there is any real data.
