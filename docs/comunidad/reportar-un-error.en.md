# Reporting a bug

Didacta is in **alpha**: reports from the people who install it are the main source of fixes. This page explains what to check before opening an issue, what data is needed and how to send it without leaking sensitive information.

Bugs are reported as a **GitHub issue** using a template:

[Open an issue :material-github:](https://github.com/va360labs/didacta-io/issues/new/choose){ .md-button .md-button--primary }

!!! danger "Is it a security problem?"

    **Do not open a public issue.** Vulnerabilities (authentication or RLS bypass, cross-tenant data leaks, RCE, SQL injection, privilege escalation…) are reported by email to **`security@didacta.io`** following the [security policy](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md). Publishing them in the open leaves every self-hosted installation exposed until there is a patch.

## Before opening the issue

1. **Check [Troubleshooting](../instalacion/solucion-de-problemas.md).** The most frequent installation failures (missing pgvector, a half-applied migration, a port already in use, a short `AUTH_SECRET`, emails stuck in Mailpit) are there with their fixes.
2. **Upgrade to the latest alpha.** During the closed alpha, only the latest published version is supported; if the bug is from an earlier alpha, you will be asked to reproduce it on the current one. See [Upgrading](../instalacion/actualizacion.md).
3. **Search the [open issues](https://github.com/va360labs/didacta-io/issues).** If it has already been reported, add your case as a comment (your version, your operating system, your logs) instead of opening a duplicate: it helps reproduce it far more.
4. **Make sure it is a bug and not a usage question.** Questions about usage, installation or architecture belong in [GitHub Discussions](https://github.com/va360labs/didacta-io/discussions), which is where they get answered.

## Which template to use

| Situation | Where it goes |
| --- | --- |
| Something does not work: an error, a blank screen, a 500, wrong data | The **🐛 Bug report** template |
| It works, but it is confusing, slow or badly organised | The **💬 General feedback** template |
| A feature is missing or you want a change in behaviour | The **✨ Feature request** template |
| A question about usage, installation or configuration | [Discussions](https://github.com/va360labs/didacta-io/discussions) |
| A security vulnerability | `security@didacta.io` (**never** an issue) |
| Commercial use, Enterprise or Cloud | `licensing@didacta.io` |

## The data we need

The bug template asks for it item by item. These are the ones that are hardest to gather afterwards:

### The exact version

Any of these three will do:

```bash
# 1. The version the API reports (no authentication needed)
curl -fsS http://localhost:4000/healthz
# → {"status":"ok","service":"api","version":"0.0.1-alpha.114",...}

# 2. The image tag you have deployed
docker compose -f docker-compose.alpha.yml images didacta

# 3. Whatever you pinned in .env
grep DIDACTA_IMAGE_TAG .env
```

### How you are running it

`docker compose -f docker-compose.alpha.yml`, `pnpm dev` locally, or something else (Kubernetes, managed Postgres, a reverse proxy in front…). If your setup differs from the [reference Docker Compose](../instalacion/docker-compose.md), say so: it changes the diagnosis considerably.

### Steps to reproduce

Numbered, from a freshly started installation if you can, and stating **which role** you are using (administrator, instructor, student). A bug that only appears with one specific role, when that is not mentioned, gets closed as not reproducible.

### Logs

```bash
# The last 200 application log lines, into a file you can attach
docker compose -f docker-compose.alpha.yml logs --no-color --tail 200 didacta > didacta-logs.txt

# Service status and healthchecks
docker compose -f docker-compose.alpha.yml ps
```

### If it is a UI bug

Open your browser's developer tools (`F12`) and add:

- The **console**: the red error, in full.
- The **Network** tab: the failing request — URL, method and response code (401, 402, 403, 500…). A **402** is not a failure: it is an [Enterprise](../enterprise/index.md) capability your license does not cover.
- A **screenshot**, which usually saves three rounds of back and forth.

## Anonymise before you paste

Logs and screenshots from a real installation carry personal data and credentials. Review them before uploading — a GitHub issue is public and indexable, and deleting it afterwards does not undo the fact that someone already read it.

| Replace | With |
| --- | --- |
| Emails and names of students, instructors or customers | `student@example.com`, `First Last` |
| JWT tokens, session cookies, API keys | `<REDACTED>` |
| `AUTH_SECRET`, database passwords, SMTP credentials | `<REDACTED>` |
| Stripe keys (`sk_live_…`, `whsec_…`) and AI provider keys | `<REDACTED>` |
| Your Enterprise license key (a signed JWT) | `<REDACTED>` |
| Internal domains or private IP addresses, if you care about them | `example.com`, `10.0.0.x` |

!!! warning "If you have already published a credential"

    Consider it compromised: rotate it in your installation (regenerate the key, change the password) as well as editing the issue. Changing `AUTH_SECRET` invalidates every open session, which is exactly what you want in that situation.

## How to write the report

- **One bug per issue.** Three bugs in a single issue get fixed at different speeds and end up blocking each other.
- **A specific title.** "The certificate is issued without the student's name when the course has two modules" is useful; "It doesn't work" is not.
- **Separate what you expected from what happened.** That is the part that decides whether the behaviour is a bug or a product decision.
- **Say whether it is always reproducible or intermittent**, and how many times out of how many. Race conditions are hunted very differently.
- **Mention what you have already ruled out.** Whether you restarted, whether you tried another browser, whether it also happens on a clean installation.

??? example "Example of a complete report"

    **Title:** Redeeming an invitation code returns 500 if the course is already archived

    **Version:** 0.0.1-alpha.114 · **Running:** `docker compose -f docker-compose.alpha.yml` · **OS:** Ubuntu 22.04

    **Steps:**

    1. As an administrator, I create a course and generate an invitation code.
    2. I archive the course from the administration listing.
    3. As a student, I go to `/canjear` and enter the code.

    **Expected:** a message saying the course is no longer available.

    **Actual:** a 500 error. In the logs: `TypeError: Cannot read properties of null (reading 'id')`.

    **Reproducible:** always, 5 out of 5, on a clean installation too.

    I am attaching `didacta-logs.txt` with the emails replaced by `@example.com`.

## What happens next

1. The report is reviewed and labelled. If some data is missing to reproduce it, it is requested in a comment — an issue with no answer to that request eventually gets closed for lack of information, and it can always be reopened.
2. If it reproduces, that is confirmed in the issue and it is prioritised by impact: data loss and failures that block installation come first.
3. When the fix is merged, the issue is closed referencing the commit, and the change ships in the next release with an entry in the [CHANGELOG](https://github.com/va360labs/didacta-io/blob/main/CHANGELOG.md).

There is no committed turnaround time for bugs during the alpha; the committed timelines are those in the [security policy](https://github.com/va360labs/didacta-io/blob/main/SECURITY.md), which apply only to vulnerabilities.

## What if you want to fix it yourself?

Community patches are welcome. Open the issue anyway — it serves as the reference for the PR — and follow the [Contributing](contribuir.md) guide: it requires signing the CLA, Conventional Commits and tests for business logic.
