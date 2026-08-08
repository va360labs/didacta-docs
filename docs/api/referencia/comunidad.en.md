# Reference — Community and people

Community, messaging, gamification, resources, surveys, referrals, theming, member registration and membership. Every route hangs off `/api/v1`.

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
| GET · POST · PATCH · DELETE | `/modules/community/spaces[/:slug]` | read Bearer · write admin | Spaces; the 4 system ones are editable but cannot be deleted (409). |
| GET | `/modules/community/members` | Bearer | Paginated directory of active members. |
| GET | `/modules/community/stats` | Bearer | The tenant's active members and courses. |
| GET · POST | `/modules/community/broadcasts` | admin | Announcements with status and batch resumption. |
| POST | `/modules/community/digest/run-now` | super_admin | Forces the weekly digest (QA) → 202. |
| GET | `/modules/community/unsubscribe?token=` | Public (HMAC token from the email) | Unsubscribes from announcements; returns HTML. |

**External API** — `/community-api` (an API key with the `community:post` scope, whose owner must be an admin): `GET /community-api/spaces` (where to publish) and `POST /community-api/posts` (publishes with `source='api'`; a non-existent `space` → 422 with the valid slugs).

**Errors**: `NOT_AUTHOR` / `NOT_MODERATOR` 403 · `NESTED_REPLIES_TOO_DEEP` / `REACTION_TARGET_MISSING` 422 · `TAG_NAME_EXISTS` / `SPACE_EXISTS` / `SPACE_NOT_DELETABLE` 409 · not found 404.

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

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET | `/modules/referrals/me` · `me/stats` | Bearer | My code and link (`/unete?ref=`) · clicks, sign-ups, commissions and history. |
| POST | `/modules/referrals/track` | Public | Records a click (deduplicated by code + day + IP hash; the IP is never stored in the clear). |
| GET · PUT | `/modules/referrals/admin/config` | admin | The program's policy: `commissionBps`, scope, windows, guarantee, minimum payout. |
| GET | `/modules/referrals/admin/commissions` · `referrers` | admin | Commissions with filters and totals · referrers with their metrics. |
| POST | `/modules/referrals/admin/commissions/:id/approve` · `revoke` | admin | Approves · revokes with a mandatory reason. |
| POST | `/modules/referrals/admin/payouts` | admin | An atomic manual payout of a batch of `APPROVED` commissions, with an external reference. |

## Theming — `/modules/theming`

| Method | Route | Auth | What it does |
|---|---|---|---|
| GET · PUT | `/modules/theming/me` | read Bearer · write admin | The tenant's theme (hue/saturation, whitelisted fonts, sign-in headlines). A non-empty `customCss`/`footerHtml` requires `feat:white_label` (402). |
| POST | `/modules/theming/me/reset` | admin | Returns to the defaults. |
| POST · DELETE | `/modules/theming/me/logo` | admin | Uploads (base64, ≤2 MB, png/jpeg/svg/webp) · deletes the logo. |
| GET | `/modules/theming/tenants/:tenantId/logo` | Public | Serves the logo (needed on `/signin` before authentication). |

## Member registration — `/modules/member-registration`

**Public flow** (tenant resolved by domain; the steps are chained with signed tickets):

| Method | Route | What it does |
|---|---|---|
| GET | `/modules/member-registration/config` | Which steps the wizard requires (`verifiers`, `botUsername`). |
| POST | `/modules/member-registration/telegram/verify` | Validates the Telegram Login Widget signature and group membership → a ticket (15 min). |
| POST | `/modules/member-registration/otp/request` · `otp/verify` | Sends the code by email · validates it → a `verificationToken` (30 min). |
| POST | `/modules/member-registration/register` | Creates the `PENDING` user and notifies the approver → `{ status: 'PENDING' }`. |
| GET | `/modules/member-registration/decision?token=` | The approve/reject link from the approver's email (302 to the result). |

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
| GET | `/membership/page` | Public | The data behind `/unete`: active plans, courses with a reference price, testimonial. |
| POST | `/membership/checkout` | Public | Anonymous subscription checkout: `{ planId, email?, referralCode? }` → `{ url, sessionId }`. |
| GET · POST · PATCH · DELETE | `/membership/admin/plans[/:id]` | admin | Plans: name, billing period (1-12 months), price in cents, struck-through price, trial, featured. Deleting a plan that has sales deactivates it instead. |
| GET · PUT | `/membership/admin/config` | admin | The public page: active, headlines, access group, lesson limit during the trial, per-course prices, testimonial. |

**Errors**: `MEMBERSHIP_PAGE_INACTIVE` 404 · `MEMBERSHIP_CONFIG_INCOMPLETE` 422 · `SUBSCRIPTIONS_STRIPE_CONFIG_MISSING` 503 · `SUBSCRIPTIONS_STRIPE_API_ERROR` 502.
