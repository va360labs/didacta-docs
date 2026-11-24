# Reference — Community and people

Community, messaging, gamification and retention, resources, surveys, referrals, theming, member registration, membership, automatic retention messages and help and suggestions. Every route hangs off `/api/v1`.

**Auth**: `Bearer` = an authenticated user · `admin` = tenant_admin/super_admin · `staff` = admin + instructor · `Public` = no session (the tenant is resolved by domain).

## Community — `/modules/community`

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/modules/community/posts` | Bearer | Creates a post: `title`, `body`, `courseId?`, `tags?` (≤10). `notifyAll` (admin only) also generates an email + bell broadcast; `important` ignores the opt-out. |
| GET | `/modules/community/posts` | Bearer | The feed with filters (`courseId`, `authorId`, `tag`, `source`, `sort: recent\|oldest\|most_commented`, `limit`). |
| GET · PATCH · DELETE | `/modules/community/posts/:id` | Bearer | Detail with comments and reactions · editing (author or admin) · soft delete (author). |
| POST | `/modules/community/posts/:id/comments` | Bearer | Comments (1 level of nested replies). |
| DELETE | `/modules/community/comments/:id` | Bearer (author) | Soft deletes the comment. |
| POST · DELETE | `/modules/community/reactions[/:id]` | Bearer | Emoji reaction on a post or comment (idempotent) · removal. |
| GET | `/modules/community/attachments` | Bearer | Attachments extracted from the posts (the gallery). |
| GET | `/modules/community/users/search?prefix=` | Bearer | Mention autocomplete (max. 8). |
| GET | `/modules/community/mentions/me` | Bearer | My most recent mentions. |
| GET · PUT | `/modules/community/me/preferences` | Bearer | Preferences (for example `digestOptOut`). |
| POST | `/modules/community/posts/:id/moderate` · `comments/:id/moderate` | admin | Hides/restores (`{ hidden, reason? }`) — reversible, and distinct from deletion by the author. |
| POST | `/modules/community/posts/:id/pin` · `unpin` | admin | Pins/unpins the post in the feed. |
| GET · POST · PUT · DELETE | `/modules/community/tags[/:id]` | read Bearer · write admin | Curated tags (`name`, hex `color`, `icon`). |
| GET · POST · PATCH · DELETE | `/modules/community/spaces[/:slug]` | read Bearer · write admin | Spaces; the 4 system ones are editable but cannot be deleted (409). `accessGroupIds` when creating or editing: only for those levels (empty = everyone). Those of a level the viewer does not have are listed with `locked: true`, name, icon and `accessLevels`, no description. |
| GET | `/modules/community/members` | Bearer | Paginated directory of active members. |
| GET | `/modules/community/stats` | Bearer | The tenant's active members and courses. |
| GET | `/modules/community/sidebar` | Bearer | The feed's sidebar in one call: the classroom card (brand, members, presence), the month's ranking and the latest comments. Each block comes `null` or empty when there is no data. |
| GET · POST | `/modules/community/broadcasts` | admin | Announcements with status and batch resumption. |
| POST | `/modules/community/digest/run-now` | super_admin | Forces the weekly digest (QA) → 202. |
| GET · POST | `/modules/community/unsubscribe` | Public (HMAC token from the email) | Unsubscribes from the community's emails: `GET ?token=` from the email link (returns HTML) · `POST` **one-click unsubscribe** (RFC 8058) from the mail client. |

**External API** — `/community-api` (an API key with the `community:post` scope, whose owner must be an admin): `GET /community-api/spaces` (where to publish) and `POST /community-api/posts` (publishes with `source='api'`; a non-existent `space` → 422 with the valid slugs).

**Errors**: `NOT_AUTHOR` / `NOT_MODERATOR` 403 · `NESTED_REPLIES_TOO_DEEP` / `REACTION_TARGET_MISSING` 422 · `TAG_NAME_EXISTS` / `SPACE_EXISTS` / `SPACE_NOT_DELETABLE` 409 · not found 404. Spaces by level: reading something from a space closed to the viewer gives 404 (as if it did not exist); posting, commenting or reacting in it, 403 `COMMUNITY_SPACE_LOCKED`; a group that does not exist in `accessGroupIds`, 422 `COMMUNITY_ACCESS_GROUP_UNKNOWN`. See [Community → Spaces by level](../../modules/catalog/community.md#spaces-by-level).

## Messaging — `/modules/messaging`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/messaging/conversations` | Bearer | The inbox: space rooms, the instructors channel and direct messages, with unread counts. |
| POST | `/modules/messaging/dm` | Bearer | Opens (or creates, idempotent per pair) the direct message with another member. |
| POST | `/modules/messaging/spaces/:slug/open` · `faculty/open` | Bearer | Opens the space's room · the private channel with the instructors (auto-provisioned). |
| GET · POST | `/modules/messaging/conversations/:id/messages` | Bearer | Cursor-paginated history (50) · sending (`body` 1-4000, quota 20/min). |
| POST | `/modules/messaging/conversations/:id/typing` | Bearer | The "typing" signal (SSE only, quota 30/min) → 204. |
| POST | `/modules/messaging/conversations/:id/read` | Bearer | Marks it as read (`lastReadAt`). |
| GET | `/modules/messaging/presence` · `members?search=` | Bearer | Live presence · member search for opening a direct message. |
| POST | `/modules/messaging/stream-ticket` | Bearer | An SSE ticket of about 60 s. |
| GET | `/modules/messaging/stream?ticket=` | Ticket | **SSE**: `message.created`, `typing`, `ping`. |

**Errors**: `MESSAGING_NOT_PARTICIPANT` 403 · `MESSAGING_SELF_DM` / `MESSAGING_STAFF_NO_FACULTY` 422 · `MESSAGING_RATE_LIMITED` 429 · non-operational account 403.

## Gamification — `/modules/gamification`

**Member (Bearer):** `GET leaderboard?range=week|month|all` · `GET me` · `GET me/history` · `GET levels` · `GET challenges` · `GET me/perks` · `POST perks/:id/request` · `POST challenges/:id/submit` (`proofUrl?`, `note?`).

**Operator:**

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · PUT | `/modules/gamification/admin/rules[/:key]` | admin | Automatic rules: points, daily cap, activation. |
| POST · PUT · DELETE | `/modules/gamification/admin/levels[/:id]` | admin | Levels (creating/editing repositions profiles). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/perks[/:id]` | admin | Level perks (per-student quota, waiting period). |
| GET · POST | `/modules/gamification/admin/perk-requests[/:id/handle]` | staff | Perk requests · handling them (`APPROVED\|DONE\|REJECTED`). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/challenges[/:id]` | staff | Challenges with a reward and a date window (deleting one does not remove points already awarded). |
| GET · POST | `/modules/gamification/admin/submissions[/:id/review]` | staff | Submissions · approving (crediting the points) or rejecting. |
| POST | `/modules/gamification/admin/backfill` | admin | Populates the ledger with historical activity (idempotent). |

**Retention and celebrations (streaks, badges, rewards and level-gated content):**

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/gamification/me/retention` | Bearer | My streak, my badges (and the catalog) and my preferences. |
| PUT | `/modules/gamification/me/prefs` | Bearer | `{ celebrate?, showStreak? }`: stay out of celebrations / do not show my streak. |
| GET | `/modules/gamification/me/rewards` | Bearer | Rewards I have received and those I still have to earn. |
| GET | `/modules/gamification/me/locks` | Bearer | Content unlocked by level, as seen by me (it does not include what I have through purchase or subscription). |
| GET | `/modules/gamification/members/summary?ids=` · `/members/:userId/summary` | Bearer | Level, badges and streak of several members (cards) · of one (their profile). |
| GET · PUT | `/modules/gamification/admin/retention/settings` | admin | Switches: `streaksEnabled`, `badgesEnabled`, `celebrateCourses`, `celebrateIntros`, `celebrateAnniversaries`, `celebrateLevels`. |
| GET · PUT | `/modules/gamification/admin/badges[/:key]` | admin | Badge catalog · rename or turn one off (`{ name?, enabled? }`). |
| GET | `/modules/gamification/admin/lockable-content` | admin | Courses (and their sections) that can be gated by level. |
| GET · PUT · DELETE | `/modules/gamification/admin/locks[/:id]` | admin | List · gates a course or a section until a level (`{ targetKind: COURSE \| MODULE, targetId, levelId, grantPaid? }`) · removes (204). |
| GET · POST · PUT · DELETE | `/modules/gamification/admin/rewards[/:id]` | admin | Automatic rewards: `{ title, triggerKind: LEVEL \| POINTS, levelId?, points?, kind: MESSAGE \| CODE, message?, url?, code?, active? }`. |
| GET | `/modules/gamification/admin/reward-grants` | admin | Log of rewards delivered. |
| POST | `/modules/gamification/admin/celebrations/anniversaries/run` | admin | Publishes this classroom's anniversaries for today right now (it does so by itself every morning). Idempotent. |

**Errors**: `GAMIFICATION_CHALLENGE_CLOSED` / `GAMIFICATION_PERK_UNAVAILABLE` / `GAMIFICATION_ALREADY_SUBMITTED` / `GAMIFICATION_ALREADY_REVIEWED` 409 · validation 422 · not found 404.

## Resources — `/modules/resources`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · POST | `/modules/resources/collections` | read Bearer · write staff | Collections (it seeds the 6 defaults) · creation with a cover image. |
| GET · PUT · DELETE | `/modules/resources/collections/:id` | staff (read Bearer) | The collection + its resources with a search · editing · deletion **only if it is empty**. |
| POST | `/modules/resources` | Bearer | Shares a resource: `collectionId`, `kind: FILE\|LINK`, `title`, `url`. |
| POST | `/modules/resources/:id/download` | Bearer | Records the download and returns the URL. |
| DELETE | `/modules/resources/:id` | author or staff | Deletes the resource. |

## Surveys — `/modules/surveys` (anonymous responses)

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/surveys/sessions/:sessionId` | Bearer | A live class's survey + whether I have already answered. |
| POST | `/modules/surveys/:id/responses` | Bearer | An anonymous response (1 per survey; deduplicated by HMAC hash, the `userId` is never persisted). |
| GET | `/modules/surveys/admin[/:id/results]` | admin | Listing · aggregate results (NPS, averages, free text). |
| POST | `/modules/surveys/admin/sessions/:sessionId` | admin | Creates the post-class survey without waiting for the Zoom webhook. |
| POST | `/modules/surveys/admin/:id/close` · `reminders/run` | admin | Closes the survey · forces the reminder sweep. |

**Errors**: `SURVEYS_CLOSED` / `SURVEYS_ALREADY_RESPONDED` 409 · `SURVEYS_INVALID_ANSWER` 422.

## Referrals — `/modules/referrals`

**Member:**

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/referrals/me` | Bearer | My code and link (`/join?ref=`), creating it if it does not exist. 404 if the program is inactive; 403 if it requires an active membership and I do not have one. |
| POST | `/modules/referrals/me/terms` | Bearer | I declare I am over 18 and accept the current version of the terms. Without this there is no link and no commissions are attributed; 403 `REFERRALS_TERMS_REQUIRED` if the version is not the current one. |
| GET · PUT | `/modules/referrals/me/payee` | Bearer | Details to be paid in cash (full name, country, NIF/NIE or tax identifier, IBAN), encrypted; GET returns them **masked** (last 4 only) or `null`. PUT: 422 `REFERRALS_PAYEE_INVALID` with the failing fields (never their values). |
| POST | `/modules/referrals/me/payouts` | Bearer | I request the cash payout of what is confirmed. It needs payout details and reaching the classroom's minimum; 409 if one is already requested or details are missing. |
| GET | `/modules/referrals/me/payouts/:id/receipt` | Bearer | PDF receipt of one of my own paid cash payouts. |
| GET | `/modules/referrals/me/stats` | Bearer | Clicks, attributed sign-ups, commissions by status, history and settlements. Works even if the program is inactive (`programActive=false`) to show the history. |
| GET | `/modules/referrals/me/invitees` | Bearer | My invitees: initials, join date, status (`trial`, `paying`, `bought_course`, `cancelled`, `joined`) and the non-voided commission generated. **Never** a name or email. |
| POST | `/modules/referrals/me/redeem-credit` | Bearer | Redeems what was earned (approved, unpaid commissions) as a discount on the next membership installment (a credit to the Stripe customer balance). Idempotent. 403 without an active membership or with the option off; 409 if there is nothing to redeem. |

**Public:**

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/modules/referrals/track` | Public | Records a click on `/join?ref=CODE` (classroom by domain; deduplicated by code + day + IP hash; the IP is never stored in the clear). Best-effort: never fails because of an invalid code or inactive program. |
| GET | `/modules/referrals/public/terms` | Public | The classroom's public program terms: current version and real figures (percentage, attribution and guarantee days, minimum payout, discount cap). |

**Administration (admin):**

| Method | Route | What it does |
|---|---|---|
| GET · PUT | `/modules/referrals/admin/config` | The program's policy: `commissionBps`, scope, windows, guarantee, minimum payout. Changes **do not recalculate** commissions already accrued (basis points are sealed per commission). |
| GET | `/modules/referrals/admin/commissions` · `referrers` | Commissions with filters and totals by status · referrers with metrics. |
| POST | `/modules/referrals/admin/commissions/:id/approve` · `revoke` | Approves a `PENDING` commission without waiting for the guarantee · revokes (`PENDING`/`APPROVED`) with a mandatory reason. |
| GET | `/modules/referrals/admin/cash-payouts` | The classroom's cash payouts (requested and paid) with what is needed to pay: name, tax identifier and IBAN. |
| POST | `/modules/referrals/admin/cash-payouts/:id/paid` | Marks a requested payout as paid, with the transfer reference: the settlement becomes `DONE` and its commissions `PAID`. |
| GET | `/modules/referrals/admin/cash-payouts/:id/receipt` | PDF receipt of a paid cash payout of the classroom. |
| POST | `/modules/referrals/admin/payouts` | An atomic manual payout of a batch of `APPROVED` commissions, with a mandatory external reference. |

## Theming — `/modules/theming`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · PUT | `/modules/theming/me` | read Bearer · write admin | The tenant's theme (hue/saturation, whitelisted fonts, sign-in headlines). A non-empty `customCss`/`footerHtml` requires `feat:white_label` (402). |
| POST | `/modules/theming/me/reset` | admin | Returns to the defaults. |
| POST · DELETE | `/modules/theming/me/logo` | admin | Uploads (base64, ≤2 MB, png/jpeg/svg/webp) · deletes the logo. |
| GET | `/modules/theming/tenants/:tenantId/logo` | Public | Serves the logo (needed on `/signin` before authentication). |
| POST · DELETE | `/modules/theming/me/favicon` | admin | Uploads (base64, ≤1 MB, png/jpeg/svg/webp) · removes the favicon. |
| GET | `/modules/theming/tenants/:tenantId/favicon` | Public | Serves the favicon (also drawn on screens with no session). |
| POST · DELETE | `/modules/theming/me/cover` | admin | Uploads (≤10 MB) · removes the academy's cover —the header of its card in the community—; without it the card goes back to the color band. |
| GET | `/modules/theming/tenants/:tenantId/cover` | Public | Serves the cover (an `<img>` does not send the token). |

## Member registration — `/modules/member-registration`

**Public flow** (tenant resolved by domain; the steps are chained with signed tickets):

| Method | Route | What it does |
|---|---|---|
| GET | `/modules/member-registration/config` | Which steps the wizard requires (`verifiers`, `botUsername`) and whether people get in straight away (`autoApprove`: free level with a verified email). |
| POST | `/modules/member-registration/telegram/verify` | Validates the Telegram Login Widget signature and group membership → a ticket (15 min). |
| POST | `/modules/member-registration/otp/request` · `otp/verify` | Sends the code by email · validates it → a `verificationToken` (30 min). |
| POST | `/modules/member-registration/register` | Creates the `PENDING` user and notifies the approver → `{ status: 'PENDING' }`. With the [free level's automatic approval](../../modules/catalog/member-registration.md#automatic-approval-with-the-free-level), it approves it straight away without notifying the approver → `{ status: 'ACTIVE' }`. |
| GET · POST | `/modules/member-registration/decision` | `GET ?token=`: the approve/reject link from the approver's email **only reads** and redirects (302) to the confirmation screen · `GET decision/preview`: what that token would do, without running it · `POST`: runs the confirmed decision. |

If the tenant requires a verifier that is not operational (for example Telegram with no bot), it returns a **fail-closed 503**.

**Administration** (admin):

| Method | Route | What it does |
|---|---|---|
| GET · POST | `/modules/member-registration/admin/requests` | Pending requests with the payment lookup · manual sign-up without OTP. |
| POST | `…/admin/requests/:userId/rerun` · `decision` | Re-runs the subscription lookup · approves/rejects from the panel. |
| GET · POST | `…/admin/requests/:userId/renewal-context` · `renewal-email` | Renewal context (the Stripe link) · sends the payment reminder. |
| GET · POST · DELETE | `/modules/member-registration/payment-flags[/:id]` | Overdue payment flags (matched by email or Telegram) + `POST …/import` for an atomic CSV load (≤5000). |

## Membership — `/membership`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/membership/page` | Public | The data behind `/join`: active plans, courses with a reference price, testimonial. With levels, also `tiers` (what each includes: courses, spaces and upcoming live classes), `freeSignupPath` and `freeSignupInstant`. |
| GET | `/membership/tiers` | Public | Levels that are on: `{ tiers: [{ id, name, accessGroupId, isFree }] }`. |
| POST | `/membership/checkout` | Public | Anonymous subscription checkout: `{ planId, email?, referralCode? }` → `{ url, sessionId }`. |
| GET · POST · PATCH · DELETE | `/membership/admin/plans[/:id]` | admin | Plans: name, billing period (1-12 months), price in cents, struck-through price, trial, featured and `tierId` (the level; required when there are paid levels, set on creation). Deleting a plan that has sales deactivates it instead. |
| GET · POST | `/membership/admin/tiers` | admin | [Levels](../../modules/catalog/subscriptions.md#levels): list (courses, prices, people in, plans without a level and the free sign-up state) · creates `{ name, description?, isFree?, allCourses?, courseIds? }` (the first paid level adopts the existing plans; the free one opens free sign-up). |
| PATCH · DELETE | `/membership/admin/tiers/:id` | admin | Edits `{ name?, description?, active?, courseIds?, manualReview? }` · deletes a level with nobody in it (with students, 409: turn it off). |
| PUT | `/membership/admin/tiers/order` | admin | Orders the levels (`{ tierIds }`): the order of the cards on `/join`. |
| GET · PUT | `/membership/admin/config` | admin | The public page: active, headlines, access group, lesson limit during the trial, per-course prices, testimonial. |
| GET | `/membership/checkout-session/:sessionId` | Public | Return from the membership payment (thank-you screen): whether **this** academy's session is completed, which plan, how much and when. |
| GET · PUT | `/membership/admin/retention` | admin | "When someone wants to leave": what is offered when cancelling (pause and maximum months, discount with % and months). |
| GET | `/membership/admin/retention/summary` | admin | Cancellation summary: how many pressed "Cancel me", how many stayed (pause or discount) and the reasons. |

**Errors**: `MEMBERSHIP_PAGE_INACTIVE` 404 · `MEMBERSHIP_CONFIG_INCOMPLETE` 422 · `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING` 503 · `SUBSCRIPTIONS_STRIPE_API_ERROR` 502. Levels: `MEMBERSHIP_TIER_NOT_FOUND` 404 · `MEMBERSHIP_TIER_INVALID` / `MEMBERSHIP_TIER_IS_FREE` 400 · `MEMBERSHIP_TIER_IN_USE` / `MEMBERSHIP_PLAN_NAME_TAKEN` 409.

## Retention — `/modules/retention` (admin)

The classroom's automatic messages (welcome, inactivity, course completed…) and the at-risk member list. Everything requires administering the classroom; with the module turned off for the tenant, **403** `TENANT_MODULES_MODULE_NOT_ACTIVE`.

| Method | Route | What it does |
|---|---|---|
| GET · PUT | `/modules/retention/admin/settings` | Who signs the messages (`senderUserId`) and the anti-spam rules (`manualCooldownHours`, `sendWindowStartHour`, `sendWindowEndHour`). |
| GET | `/modules/retention/admin/templates` | Ready-made templates, in the classroom's language. |
| POST | `/modules/retention/admin/templates/:key/activate` | Activates a template in one click (or turns it back on if it exists). |
| GET · POST | `/modules/retention/admin/automations` | Automatic messages with their metrics · creates one: `{ trigger, name?, days?, resendAfterDays?, body, emailEnabled?, emailSubject?, enabled? }`. |
| PATCH · DELETE | `/modules/retention/admin/automations/:id` | Edits, turns on or off · deletes (204; its metrics are kept). |
| POST | `/modules/retention/admin/preview` | Preview with the real data of a person or of the viewer: `{ trigger, body, subject?, userId? }`. |
| POST | `/modules/retention/admin/run` | Runs **this** classroom's sweep now, without waiting for the 5 minutes (same anti-spam rules). |
| GET | `/modules/retention/admin/at-risk` | At-risk members (`?reason=inactive\|paid_not_started\|stalled`, `days` 1–365, `search`): not signing in, paid and did not start, half-way through. |
| POST | `/modules/retention/admin/at-risk/:userId/recovery` | Adds the person to the classroom's recovery message. |

Triggers (`trigger`): `WELCOME`, `DAYS_AFTER_JOIN`, `INACTIVE`, `COURSE_COMPLETED`, `FIRST_POST`, `LEVEL_UP` and `ANNIVERSARY`.

## Help and suggestions — `/modules/soporte`

Tickets for "something is not working / I have a problem / I have a suggestion". **The destination is decided by the server from the session's roles**, never the body: a member writes to the **classroom team** (destination `AULA`); whoever administers the classroom writes to **Didacta** (destination `PLATAFORMA`, notifying the operator email defined by `SUPPORT_OPERATOR_ALERT_EMAIL`). With the module turned off, 403 `TENANT_MODULES_MODULE_NOT_ACTIVE`.

| Method | Route | Auth | What it does |
|---|---|---|---|
| POST | `/modules/soporte/tickets` | Bearer | Opens a ticket: `{ tipo: FALLO \| PROBLEMA \| MEJORA, texto (5–4000), contexto?, captura? }`. The screenshot is a PNG, JPEG or WebP of up to 5 MB in base64. **At most 5 tickets in 24 hours.** It answers as soon as it is saved; notifications go out afterwards. |
| GET | `/modules/soporte/canal` | Bearer | Who a ticket from the caller would go to (`AULA` or `PLATAFORMA`) and whether anyone is on the other side; the web does not show the button otherwise. |
| GET | `/modules/soporte/tickets/mios` | Bearer | My tickets and their status, newest first. |
| GET | `/modules/soporte/tickets/:id/captura` | author or admin | A ticket's screenshot (404 to anyone else). |
| GET | `/modules/soporte/admin/tickets` | admin | The classroom team's inbox: the 200 newest, with `?estado=` and `?tipo=` filters, and how many there are in each state. |
| GET | `/modules/soporte/admin/menu` | Bearer | `{ allowed, badge }`: whether the caller sees the inbox and how many new tickets are untaken. |
| GET · PATCH | `/modules/soporte/admin/tickets/:id` | admin | A ticket with its author, context and internal note · changes `estado` (`NUEVO`, `EN_CURSO`, `RESUELTO`, `DESCARTADO`) or `notaInterna` (at least one); if the state changes, whoever opened it is notified. |
