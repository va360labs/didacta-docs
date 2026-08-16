# Selling and integrating from outside Didacta

This page is for when **the shop, the website or the CRM live elsewhere** and Didacta provides the classroom: a WordPress site that wants to stop rendering its course pages with LearnDash, a custom CMS that charges with its own Stripe, an n8n flow that enrols people when an order comes in.

Two complementary APIs, both authenticated with a tenant API key:

| | What for | Scopes |
|---|---|---|
| **`/inscribe`** — write | Grant and revoke access after a payment you collected | `enrollments:write`, `courses:read` |
| **`/integrations`** — read | Render the course page with real data and know whether the visitor already bought | `courses:read`, `enrollments:read` |

## First things first: this is called from your server

**Didacta's API does not enable CORS.** This is not an oversight waiting to be fixed — it is the shape of the integration.

- From the browser, the calls fail.
- And even if they didn't, you wouldn't want them to: **the API key grants access to the whole tenant**. Shipping it in your site's JavaScript is handing it over.

Everything on this page is called from **your site's backend**: the WordPress `functions.php`, your CMS route handler, the n8n HTTP node.

!!! warning "The two doors don't have the same capacity"
    The rate limit is **per organisation** (100/min on Community, tunable with `RATE_LIMIT_COMMUNITY_AUTH_PER_MIN`). But the **public** endpoints — the unauthenticated sales catalogue — share **a single 30/min budget for the entire installation**, across every organisation and every visitor.

    If your site renders course pages on every visit, **use the API-key endpoints**, not the public ones. On the public ones you compete with the anonymous traffic of the whole server.

## Creating the key

In **Administration → API keys**, «New key». Pick only the scopes you will use; the plaintext token is shown **once**.

```bash
curl -H "Authorization: ApiKey lmsk_..." \
     https://your-classroom.example.com/api/v1/integrations/courses
```

Note the scheme: `ApiKey`, not `Bearer`.

### Why `enrollments:read` is separate

`enrollments:read` allows asking about **any email in the organisation**. Something that only needs to render a sales page has no business querying your entire student base, or the other way round. If your integration only reads the catalogue, give it `courses:read` and nothing else.

## Reading: rendering the course page outside Didacta

### Finding the course

```bash
# By slug
GET /api/v1/integrations/courses?slug=claude-code-course

# Or by the id it had in the LMS you imported from
GET /api/v1/integrations/courses?externalId=1234&externalSource=learndash
```

The `externalId` + `externalSource` filter is what saves you copying UUIDs by hand: if the courses came in through the LearnDash importer, each one remembers which WordPress post it came from, and your plugin can resolve the mapping on its own.

### The full course record

```bash
GET /api/v1/integrations/courses/{courseId}
```

Returns everything a sales page needs: description, image, featured video, instructor, whether it issues a certificate, totals (modules, lessons, minutes), **the full syllabus module by module** and, if the course has a price in Didacta, its **purchase options** with amount and compare-at price.

!!! info "It never returns lesson content"
    The syllabus is shown to sell; the class is taught in Didacta. `content` never leaves this API under any circumstances, with any set of scopes.

A lesson may come back with `scheduled: true`: it exists in the syllabus but has a future publication date. Render it as "coming soon" rather than hiding it.

### Has this visitor already bought?

```bash
GET /api/v1/integrations/learners/state?email=ana@example.com&courseId={uuid}
```

This is the question every external course page asks. The answer decides which button you render:

| Response | What it means | What to render |
|---|---|---|
| `known: false` | That email belongs to nobody in your organisation | Sales mode |
| `hasAccess: true` | Enrolled and able to enter | "Continue where you left off" with `nextLesson.url` |
| `hasAccess: false` with `status: "PAUSED"` | **Suspended student**, typically for non-payment | Neither sales nor access: tell them to sort the payment out |

That last case is the one people get wrong. `PAUSED` is **not** "never bought": their progress is intact and they'll be back as soon as the payment clears. Treat them as a visitor and you're offering to sell something they already paid for.

## Writing: granting access after you charged

```bash
curl -X POST https://your-classroom.example.com/api/v1/inscribe \
  -H "Authorization: ApiKey lmsk_..." \
  -H "Content-Type: application/json" \
  -d '{
        "email": "ana@example.com",
        "name": "Ana Pérez",
        "courseIds": ["3f4b2c10-1a2b-4c3d-9e8f-0a1b2c3d4e5f"],
        "externalRef": "order_12345"
      }'
```

Creates the user if missing (they get a temporary password by email and must change it on first login) and enrols them. **It is idempotent**: repeating the same call duplicates nothing, so you can retry freely from your webhook queue.

### Bundles: one access group, not N enrolments

If you sell a bundle of several courses, **do not send N `courseIds`**. Create a `MULTI_COURSE` access group in Didacta and enrol against it:

```json
{ "email": "ana@example.com", "accessGroupIds": ["<bundle-uuid>"] }
```

The difference matters when the bundle changes: with a group, everyone who already bought sees the new course without your shop doing anything. With individual enrolments, access froze on the day of purchase.

`GET /inscribe/access-groups` lists the groups with their `kind` and how many courses each grants.

### Refunds: wire the revoke on day one

```bash
POST /api/v1/inscribe/revoke
{ "email": "ana@example.com", "courseIds": ["..."], "reason": "refund" }
```

This is the half everyone leaves for later and never comes back to. Without it, whoever asks for their money back keeps the course.

It is deliberately conservative: it **only cancels API-origin enrolments**. If that person is also a member or has the course through an access group, that access is **left alone** — you refunded their purchase, you didn't take away what they already had by another route.

## Being told what happens inside Didacta

If, on top of writing, you want Didacta to notify you, configure an **outbound webhook** in Administration → Webhooks. The events that matter to an external shop:

| Event | When |
|---|---|
| `billing.order.completed` | A single course was paid for **inside Didacta** |
| `billing.order.refunded` | That purchase was refunded → revoke access in your system |
| `learning.enrollment.created` | Someone got enrolled, by whatever route |
| `learning.course.completed` | They finished the course (for your congratulations email, your CRM…) |

Your endpoint receives a signed POST. **Deduplicate on the `X-Didacta-Delivery` header**: on a retry, the same event can arrive twice.

## The three mistakes everyone makes

1. **Calling from the browser.** There is no CORS, and the key would be published. Always from your server.
2. **Treating `PAUSED` as "hasn't bought".** It's a suspended student with intact progress.
3. **Not wiring the refund.** `POST /inscribe` without its `revoke` is half an integration.

## See also

- [Authentication](autenticacion.md) — creating and revoking API keys, available scopes.
- [Reference: core and cross-cutting](referencia/nucleo.md) — the full route table.
- [Access groups](../modulos/catalogo/access-groups.md) — how a bundle is put together.
- [SSO from WordPress](../modulos/catalogo/wp-sso.md) — so the buyer enters the classroom without identifying themselves again. The mechanism is a JWT signed with a shared secret: **it works for any external site**, not just WordPress.
