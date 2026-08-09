# mod.gamification — Points and challenges

<span class="didacta-chip didacta-chip--community">Community</span> · **Engagement** category (deliberately optional)

## What it does

It turns community activity into an **auditable points ledger**, with two deliberately distinct layers:

- **Activity** — automatic entries triggered by events on the bus (post, comment, resource, course completed, referral…), with low weights and a per-rule **daily cap**: they reward consistency, not volume.
- **Milestones** — **challenges** with mandatory proof and human review, carrying high weights.

On top of that, **levels** are computed on lifetime points and never go down; the **leaderboard** is computed over a date range (week/month/all time) and does move. A level unlocks **perks** defined by the operator (a 1:1 session, an extra class…) with a per-student quota and an optional waiting period; the student requests them and a person handles them.

## How it works

- The only real defence against double-crediting is the `(tenant, user, sourceKey)` uniqueness constraint, where `sourceKey` identifies the **fact** (`community.post:<id>`), not the event — the bus delivers at least once.
- **Perks never touch access groups**: access to courses is a paid product and cannot be earned by participating (an explicit product decision).
- Moderating content (hiding it) **revokes** the points it granted; restoring it gives them back.
- The **backfill** populates the ledger with historical activity: idempotent, repeatable, excluding hidden content and API posts, and it does not apply the daily cap retroactively.
- Levels and challenges **start empty on purpose** (their names and rewards are branding decisions); only the weighting rules are seeded.

## Dependencies

Optional (read-only, for the backfill): `mod.community`, `mod.learning`, `mod.resources`.

## Data model

`mod_gamification_ledger_entry` (an entry, append-only except for revocation) · `_profile` (materialised balance) · `_rule` · `_level` · `_perk` · `_perk_request` · `_challenge` · `_submission` (one submission per challenge and person).

## API

Prefix `/modules/gamification` (member + `/admin`). Details in [Reference → Community and people](../../api/referencia/comunidad.md#gamification-modulesgamification).

## Events

- **Emits**: `gamification.points.awarded/revoked`, `gamification.level.changed`, `gamification.challenge.submitted/reviewed`, `gamification.perk.requested/handled`.
- **Consumes**: `community.post.created/comment.created/post.hidden/unhidden/comment.hidden/unhidden`, `resources.resource.created/deleted`, `learning.course.completed`, `referrals.referral.attributed`.

## Configuration

Everything is per tenant, under Administration → Points & challenges (weights, caps, levels, perks, challenges). No environment variables.
