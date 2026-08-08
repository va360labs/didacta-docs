# mod.community — Community

<span class="didacta-chip didacta-chip--community">Community</span> · **Core** category (always active)

## What it does

The tenant's social feed: **posts** (optionally tied to a course and a space), **comments** with one level of replies, emoji **reactions**, `@user` **mentions** with autocomplete, admin-**curated tags** (name, color, icon) and manageable **spaces** (the 4 system ones are editable but cannot be deleted). On top of that: moderation, pinned posts, a member directory, a weekly email digest and **announcements** (broadcasts) with an unsubscribe link.

## How it works

- A deliberate distinction between **moderation hiding** (`hidden`, reversible, the row is preserved along with the reason) and **deletion by the author** (soft delete).
- A post with `notifyAll` (admin only) also generates an email + bell broadcast to every member; `important` ignores the recipient's opt-out.
- Broadcasts are sent **in throttled batches** (to protect SMTP reputation) and carry a resume cursor for retries.
- Publishing over the **external API** (`/community-api`, with an API key scoped `community:post` whose owner is an admin): those posts come in with `source='api'`, which is how automated content is distinguished.
- The weekly digest is scheduled with `COMMUNITY_DIGEST_CRON` (Mondays at 09:00 UTC by default) and honours each user's opt-out.

## Dependencies

Optional: `mod.courses` (posts tied to a course).

## Data model

`mod_community_post` · `_comment` · `_reaction` · `_mention` · `_tag` · `_space` · `_broadcast` (with status and cursor) · `_user_pref` (preferences; today, the digest opt-out).

## API

Prefix `/modules/community` + `/community-api` (integrations) + public token-based unsubscribe. Details in [Reference → Community and people](../../api/referencia/comunidad.md#community-modulescommunity).

## Events

**Emits**: `community.post.created`, `community.comment.created`, `community.reaction.added` (+ mentions). It consumes none. Its events feed [gamification](gamification.md) and the [outgoing webhooks](../../api/convenciones.md#outgoing-webhooks).

## Configuration

Spaces, tags and the digest are configured from the panel. Workers: `COMMUNITY_DIGEST_CRON`, `COMMUNITY_BROADCAST_BATCH_SIZE` (5), `COMMUNITY_BROADCAST_INTERVAL_MS` (10 s).
