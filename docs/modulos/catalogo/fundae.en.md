# mod.fundae — Fundae compliance

<span class="didacta-chip didacta-chip--community">Community</span> · **Compliance** category (can be disabled)

## What it does

Regulatory compliance for **subsidised training in Spain** (Royal Decree 694/2017):

- **Training actions** with a code, delivery mode (`PRESENCIAL`/`TELEFORMACION`/`MIXTA`), hours, dates and content blocks.
- **Subsidised companies** with a validated tax ID (DNI/NIE/CIF checksum), social security account code (CCC) and annual credit.
- **RLPT notices** to the workers' legal representatives, with the statutory 15 calendar day period calculated automatically and the evidence (PDF) stored in the Evidence Vault with its hash.
- **Subsidised groups** with a lifecycle (`DRAFT → ACTIVE → CLOSED/CANCELLED`), allocated costs and named participant enrollment.
- **Completion** with a configurable threshold (75% by default): APTO / NO_APTO / EN_CURSO per participant, with a `preview` mode.
- **Exports**: group start and group end XML files, plus a complete **audit ZIP** (SHA-256 manifest + XMLs + CSVs + RLPT evidence) verifiable offline with a dependency-free tool.

## How it works

- Starting a group validates the **RLPT lead time** (15 days) and the company's **available credit**; closing the group debits the credit consumed.
- A company's tax ID is **immutable** once created (a change of tax ID means a new company, for traceability).
- The block hours cannot add up to more than the action's hours (Fundae checks this when the XML is uploaded).
- It is the natural consumer of the `courses.publish.validate` hook: it can block a course from being published if objectives or duration are missing.
- There are role-specific views: admin, **auditor** (read-only, redacted) and `empresa_manager` (their own employees only). The instructor role has no access to Fundae.

## Dependencies

Optional: `mod.courses` (validating the linked course) and `mod.learning` (enrollments and completion).

## Data model

`mod_fundae_action` · `_block` · `_company` · `_rlpt_notice` · `_group` · `_cost` · `_group_participant`.

## API

Prefix `/modules/fundae` (actions, exports) + the admin surface at `/admin/fundae/*` (companies, RLPT, groups, costs, participants). Details in [Reference → Learning](../../api/referencia/aprendizaje.md#fundae-modulesfundae-all-admin-the-instructor-role-has-no-access) and [Reference → Administration](../../api/referencia/administracion.md#fundae-admin-the-instructor-role-has-no-access).

## Events

**Emits** 24 events covering the whole cycle: `fundae.action.*`, `fundae.company.*`, `fundae.rlpt.notice.*`, `fundae.group.*` (including XML and ZIP generation), `fundae.cost.*`, `fundae.export.generated`. It consumes none.

## Configuration

The completion threshold is configurable per group (75% by default). No environment variables of its own.
