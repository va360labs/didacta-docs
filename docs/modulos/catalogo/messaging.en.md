# mod.messaging — Messaging

<span class="didacta-chip didacta-chip--community">Community</span> · **Community** category (can be disabled)

## What it does

Native messaging with three conversation types:

- A **chat room for every community space**.
- **Direct messages** between members.
- An **automatic private channel** between each student and the instructor team (the staff enquiries inbox).

With **real-time delivery over SSE**, presence, a typing indicator, read receipts, an unread counter and rate limiting (20 messages/min, 30 typing signals/min).

## How it works

- Uniqueness is enforced in the database: one conversation per space, one per student in the instructors channel, and one per pair of users in DMs (a canonical ordered key), which makes *get-or-create* **idempotent**.
- The author's name is stored as a **snapshot** on send (it survives renames and deletions, and avoids N+1 queries).
- Deleted messages keep their body for auditing and render as "Message deleted".
- The SSE stream opens with an **ephemeral ticket** (~60 s) because `EventSource` does not accept headers; events: `message.created`, `typing`, `ping`.
- A suspended or deleted account cannot use messaging even if its token is still valid (checked against the database on every call).
- Rate limiting uses Redis and degrades to a local in-memory counter if Redis fails.

## Dependencies

Optional: `mod.community` (reading spaces for the rooms).

## Data model

`mod_messaging_conversation` (typed SPACE/DM/FACULTY, with a denormalised `lastMessageAt`) · `mod_messaging_participant` (with `lastReadAt`) · `mod_messaging_message` (author snapshot, attachment support, soft delete).

## API

Prefix `/modules/messaging` (+ the SSE stream). Details in [Reference → Community and people](../../api/referencia/comunidad.md#messaging-modulesmessaging).

## Events

**Emits**: `messaging.conversation.created`, `messaging.message.sent`. It consumes none.

## Configuration

No per-tenant configuration and no variables of its own. It needs Redis for realtime and presence.
