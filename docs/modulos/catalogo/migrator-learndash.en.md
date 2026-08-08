# mod.migrator-learndash — Migrator from WordPress + LearnDash

<span class="didacta-chip didacta-chip--community">Community</span> · **Migration** category · **Beta** · Distributed as a **signed ZIP** (third-party format)

## What it does

It migrates a whole **WordPress + LearnDash** academy to Didacta: courses, lessons, topics, quizzes, questions, users, groups, enrollments, media and progress. It is driven by a **6-step wizard** (welcome → connect with a URL + Application Password → source summary → options → *dry-run* preflight check → the real migration with live progress), designed so a non-technical administrator can complete it in an hour.

## How it works

- An **ETL pipeline with staging**: it extracts into staging tables, transforms and loads with **idempotent mapping** by checksum — re-running it duplicates nothing.
- Data errors go to a **DLQ** without aborting the job; at the end there are reconciliation reports (JSON/CSV/signed PDF for an auditor) and a verifiable SHA-256 audit chain.
- It **only reads** from the source: the WordPress site is left untouched.
- Passwords: by default it does not import hashes (PHPass is not compatible with Argon2id); every user gets an activation email to set their own.
- The job runs on the backend (you can close the browser), with a 6-hour timeout.
- It is the **real-world example of the third-party format**: it is installed as a ZIP under Administration → Module marketplace, runs in the isolated VM with sandboxed `ctx.db`/`ctx.http`/`ctx.secrets`, and its WordPress credentials are stored AES-256-GCM encrypted with a key scoped to the job.

## Dependencies

Required: `mod.courses`, `mod.learning`, `mod.assessments`. Optional: `mod.certificates`, `mod.community`. If a required module is missing, the wizard blocks the start with a clear notice.

## Data model

13 `mod_migrator_learndash_*` tables: `_jobs`, `_mappings` (source ↔ target per entity), `_dlq`, `_audit_events` (append-only, chained), `_validation_reports` and 10 staging tables (`_stg_users`, `_stg_courses`, `_stg_lessons`, `_stg_quizzes`…).

## API

Prefix `/modules/migrator-learndash`: `preflight`, `jobs` (create, status, SSE progress, cancel), reports and audit verification.

## Configuration

No environment variables: everything travels in the job the wizard creates.

!!! warning "MVP limitations"
    Automatic rollback exists but has not been validated for production — the supported path is restoring from the `pg_dump` you took beforehand. Fine-grained quiz attempt history and already-issued certificates are not migrated (templates only).
