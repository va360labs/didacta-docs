# mod.certificates — Certificates

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

Issuing and managing PDF certificates on course completion: **customisable templates** per organization (logo, copy, color, signatory) with a default template, **automatic issuing** on receiving `learning.course.completed`, unique sequential numbering per tenant, and **public verification** through a shareable link (the URL you add to a LinkedIn profile).

## How it works

- Every issued certificate stores an **immutable snapshot** (student, course, date) and the SHA-256 hash of the PDF: renaming the course afterwards does not alter an already issued certificate. The PDF is regenerated on demand from the snapshot.
- Two uniqueness constraints: `(tenant, number)` guarantees sequential numbering and `(tenant, enrollment)` prevents issuing twice for the same enrollment.
- **Public verification** (`/modules/certificates/verify/:id`) returns `{ number, studentName, courseTitle, issuedAt, valid }` without ever exposing an email address or internal data, and it accounts for revocation with an audit log entry.
- The template editor has a **PDF preview** with sample data, persisting nothing.

## Dependencies

- Hard: `mod.learning` (issuing is triggered on completion). Optional: `mod.courses` (reading course data when issuing).

## Data model

| Table | What it stores |
| --- | --- |
| `mod_certificates_template` | Templates: body, color, logo, signatory, default flag. |
| `mod_certificates_issued` | Issued certificates: number, SHA-256 hash, JSON snapshot, storage key, revocation. |

## API

Prefix `/modules/certificates`: the student's own (`me`, detail, download), templates (instructor and above) and public verification. Details in [Reference → Learning](../../api/referencia/aprendizaje.md#certificates-modulescertificates).

## Events

- **Emits**: `certificates.issued`, `certificates.revoked`.
- **Consumes**: `learning.course.completed` (automatic issuing).

## Configuration

No variables of its own; all customisation goes through the tenant's templates. Deleting a template that is in use or marked as the default returns 409.
