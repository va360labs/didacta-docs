# mod.gamification — Points and challenges

<span class="didacta-chip didacta-chip--community">Community</span> · **Engagement** category (deliberately optional)

## What it does

It turns community activity into an **auditable points ledger**, with two deliberately distinct layers:

- **Activity** — automatic entries triggered by events on the bus (post, comment, resource, course completed, referral…), with low weights and a per-rule **daily cap**: they reward consistency, not volume.
- **Milestones** — **challenges** with mandatory proof and human review, carrying high weights.

On top of that, **levels** are computed on lifetime points and never go down; the **leaderboard** is computed over a date range (week/month/all time) and does move. A level unlocks **perks** defined by the operator (a 1:1 session, an extra class…) with a per-student quota and an optional waiting period; the student requests them and a person handles them.

In addition, an optional **retention layer** so students come back: **content that unlocks by level**, **automatic rewards** on reaching a level or a number of points, **badges**, day and week **streaks**, and **celebrations** in the community.

## How it works

- The only real defence against double-crediting is the `(tenant, user, sourceKey)` uniqueness constraint, where `sourceKey` identifies the **fact** (`community.post:<id>`), not the event — the bus delivers at least once.
- **Perks never touch access groups**: access to courses is a paid product and cannot be earned by participating (an explicit product decision). The only exception is unlocking a paid course by level with **"Give it away when they reach the level"**, which the operator has to tick on purpose (see below).
- Moderating content (hiding it) **revokes** the points it granted; restoring it gives them back.
- The **backfill** populates the ledger with historical activity: idempotent, repeatable, excluding hidden content and API posts, and it does not apply the daily cap retroactively.
- Levels and challenges **start empty on purpose** (their names and rewards are branding decisions); only the weighting rules and the badge catalogue are seeded.
- **Unlock by level**: a whole course, or one of its sections, shows with a padlock ("Unlocks at level N", with the points still missing) until the person reaches the level; then it opens by itself and they are notified. A level already reached is not closed again even if a revocation lowers the points. If the course is **paid**, reaching the level does not give it away (whoever bought it always gets in) unless the operator ticks **"Give it away when they reach the level"**.
- **Streaks**: consecutive days and weeks with activity (posting, commenting or progressing in a course), counted on each person's local day. A day without activity does not break the streak until it ends, and nobody is ever warned that a streak is about to be lost.
- **Badges**: earned automatically; the product ships six (first post, first comment, first course finished, 7-day streak, 30-day streak and level 3), which the operator can disable or rename. They appear on the profile and the member card.
- **Automatic rewards**: a message or link, or a discount code the operator already created in their payments account (Didacta does not create coupons), delivered on its own when reaching a level or a number of points, with a notification and a delivery log. On academies that collect with platform payments ([mod.payouts](payouts.md)) there is no discount-code reward (those payments do not accept coupons).
- **Celebrations**: automatic posts, in the name of whoever administers the academy and in the General space, when someone finishes a course, introduces themselves, reaches a community anniversary or levels up; if there are several of the same kind on the same day they go out as a single post. Each member can choose not to appear.

!!! note "Screenshots in simple mode"
    The screenshots on this page come from an academy in **simple mode** (tabs at the top). Screens without a tab of their own live under **"More"**, with the same name as in the administration menu; in **full mode** they open from the side menu. Routes (`/admin/…`, `/studio/…`) are the same in both modes.

## Configuration

- **Activation** — `Administration → Settings`, "Modules" tab (`/admin/settings`): the toggle for "Puntos y retos" (`mod.gamification`). Disabling it removes "Leaderboard" and "Challenges" from the menu, and "Points & challenges" from the admin area.
- **A single panel** — `/admin/gamification` ("Points and challenges", People & access group of the admin area), with 7 tabs: "Challenges", "Submissions", "Levels", "Perks", "Requests", "Rules" and "Rewards and unlocks".
- **Scoring rules** — "Rules" tab ("How much each action is worth"): for each action ("Publish a post", "Reply to someone", "Share a resource", "Finish a course", "Bring a referral") you set "Points", "Cap/day" (0 = no limit) and the "Active" checkbox. Rules come seeded with default weights.
- **Backfilling history** — same "Rules" tab, the "Backfill the history" card with the "Backfill now" button: awards points for prior activity with its original date; it can be repeated without duplicating anything.
- **Rewards and unlocks** — "Rewards and unlocks" tab (the retention layer): "Content that unlocks by level" (pick "Course", "Section", "Unlocks at" and "Lock until that level"; "Open now" removes the padlock), "Automatic rewards" (name, "Delivered when they reach" a level or a number of points, "What they get": a message or link, or a code; "Create reward", with "Pause"/"Activate" and a delivery log), "Badges" ("Award badges" and rename), "Streaks" ("Count streaks") and "Celebrations in the community" (four switches: courses, introductions, anniversaries and levels). **Streaks and badges start on; the four celebrations start off.**
- **Roles** — rules, levels, perks, the backfill and the whole retention layer (rewards, unlocks, badges, streaks and celebrations): `super_admin` / `tenant_admin` only; challenges, submissions and perk requests: also `formador` (instructor).
- Environment variable: `GAMIFICATION_ANNIVERSARY_CRON` — cron of the daily anniversary sweep for the celebrations (`0 8 * * *`, UTC). No feature requires an Enterprise license.

![The points panel: the Rules tab with weights, daily caps and the history backfill](../../assets/modules/gamification/en/gamification-1-rules.png)

!!! note "The screenshot predates the "Rewards and unlocks" tab"
    The screenshot shows the panel with six tabs; the seventh, "Rewards and unlocks", was added later and does not appear in it.

## Step-by-step usage

**The operator:**

1. Review the weights and caps in "Rules"; they come seeded, but how much each action is worth is your call.
2. Build the ladder in "Levels": "New level" with "Name", "From (points)" and "Benefit". Until the first one exists nobody has a level — points accumulate all the same.
3. Hang "Perks" from each level: "What it unlocks", "Level that unlocks it", "Times per student (0 = no limit)" and "Wait between requests (days)". A perk can be paused ("Pause") without deleting it.
4. Publish challenges in "Challenges": title, "Points", "What needs to be done" and the "Require proof (screenshot, file or link) to be able to submit" checkbox. They start as a "Draft": press "Open" when the community should see them.
5. Review submissions in "Submissions" ("To review" tab): "View proof", an optional comment and "Approve and award points" or "Reject". The comment is shown to the submitter.

    ![Challenge submissions awaiting review, with the attached proof](../../assets/modules/gamification/en/gamification-2-submissions.png)

6. Handle perk requests in "Requests": "Approve" the request, fulfil it in the real world and "Mark as done" (or "Reject" with an optional reply).
7. If the community was already active before points existed, "Backfill now": the summary reports how many new entries came from posts, comments, resources, courses and referrals.

**The student:**

1. `/challenges` ("Challenges": in simple mode, the "Challenges" button on the "Leaderboard" tab; in full mode, the People group): the header says how many points they have, how many are missing for the next level and which challenge gets them there fastest ("Submit the quickest challenge"). Below: "Your ladder" with the levels, "What you can spend your points on" with the perks, and "Open challenges".

    ![The student's challenges view: points, the level ladder and open challenges](../../assets/modules/gamification/en/gamification-3-challenges.png)

2. "Submit" opens the submission modal: what they did, the proof (file or link, mandatory if the challenge requires it) and a confirmation — each challenge can be submitted **only once**. It stays "Under review" until the team resolves it.
3. With a perk unlocked, "Request it" creates the request; the screen shows its state ("Requested", "Approved", "Done") and when it can be requested again if there is a waiting period.
4. On `/challenges` they also see **"Your streak"**, **"Your badges"**, **"Your rewards"** (with their code or link and what is missing for the next one) and **"Your privacy"**: they can choose **"Appear in community celebrations"** and **"Show my streak on my profile"**. Their level, badges and streak also show on their profile (`/u/<id>`).
5. `/leaderboard` ("Leaderboard"): the ranking for "This month", "This week" or "All time", with "Your points", "Your rank" and "Your level" on top.

    ![The community leaderboard by date range](../../assets/modules/gamification/en/gamification-4-leaderboard.png)

## Permissions

`gamification.read`, `gamification.submit` and `gamification.manage`.

## Dependencies

Optional (read-only, for the backfill): `mod.community`, `mod.learning`, `mod.resources`.

## Data model

`mod_gamification_ledger_entry` (an entry, append-only except for revocation) · `_profile` (materialised balance) · `_rule` · `_level` · `_perk` · `_perk_request` · `_challenge` · `_submission` (one submission per challenge and person). Retention layer: `_settings` · `_member_prefs` · `_activity_day` · `_streak` · `_badge` · `_user_badge` · `_content_lock` · `_content_unlock` · `_reward` · `_reward_grant` · `_celebration` · `_celebration_mark`.

## API

Prefix `/modules/gamification` (member + `/admin`). Details in [Reference → Community and people](../../api/reference/community.md#gamification-modulesgamification).

## Events

- **Emits**: `gamification.points.awarded/revoked`, `gamification.level.changed`, `gamification.challenge.submitted/reviewed`, `gamification.perk.requested/handled`, `gamification.badge.awarded`, `gamification.reward.granted`, `gamification.content.unlocked`.
- **Consumes**: `community.post.created/comment.created/post.hidden/unhidden/comment.hidden/unhidden`, `resources.resource.created/deleted`, `learning.course.completed`, `learning.progress.updated`, `referrals.referral.attributed`, and its own `gamification.level.changed` and `gamification.points.awarded` (for rewards, unlocks and celebrations).
