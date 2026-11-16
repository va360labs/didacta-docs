# AI agents and the MCP server

Every Didacta classroom ships an **MCP server** (Model Context Protocol) at `POST /api/v1/mcp`. It is the "agent gateway": an AI agent (Claude Code, Codex, Hermes, Grok, Cursor, GitHub Copilot in VS Code…) connects with a key of its own and does, inside your classroom, what you would otherwise do by hand: **create and edit courses and lessons, schedule live classes and post in the community**.

It is not a second business API: each tool calls the same module service the panel uses (same validation, same text sanitising) and acts **on behalf of the administrator who created the key**.

!!! info "What it can and cannot do"
    **It can** create and edit courses and lessons, schedule live classes and post in the community.

    **It cannot**, and no scope allows it: delete anything, add or invite people, set prices, publish a course for sale or email all members. Courses an agent creates are left as **drafts**; you publish them, with their price, from "Start here". Live classes, community posts and lessons added to a course that is **already published**, on the other hand, are visible straight away: confirm with the agent before asking for those.

## Creating an agent key

There are two ways. Both require you to be an **administrator** of the classroom (`tenant_admin` or `super_admin`).

=== "From the panel (the usual way)"

    On the classroom's management screens —**Courses** (`/studio/courses`), a course editor and **Live classes** (`/studio/live`)— the **"Do it faster with your AI agent"** panel appears:

    1. **Create your connection key**: press "Create key". The token (`lmsk_…`) is shown **only once**.
    2. **Connect your agent**: pick its tab and copy the command or config block, which already carries your classroom URL and your key.

    Under the hood the panel calls `POST /api/v1/agent/keys`. That route only accepts a **person's session** (not an API key: an agent cannot mint new keys for itself), the body only takes an optional `name`, and **the server fixes the scopes**: always the four `agent:*` ones.

=== "Through the API"

    With an administrator session you can create the key at `POST /api/v1/admin/api-keys`, choosing the `agent:*` scopes yourself (and, optionally, `expiresAt`):

    ```bash
    curl -X POST https://your-classroom.example.com/api/v1/admin/api-keys \
      -H "Authorization: Bearer <accessToken>" \
      -H "Content-Type: application/json" \
      -d '{"name": "Course agent", "scopes": ["agent:courses:read", "agent:courses:write"]}'
    ```

    The response carries the plaintext `token`, once.

Agent keys are listed and **revoked** in **Administration → API keys**, like any other. Only the token's SHA-256 is stored; the list shows the creation date and the last use.

!!! warning "The key acts as you"
    Whatever the agent creates is attributed to the person who generated the key. If that person loses the administrator role (or the account stops being active), **the key stops working at that very moment**.

## Endpoint and transport

| | |
|---|---|
| **URL** | `https://your-classroom.example.com/api/v1/mcp` (your classroom's domain) |
| **Transport** | MCP "Streamable HTTP", **stateless**: each `POST` carries one JSON-RPC 2.0 message and is answered with `application/json` |
| **Sessions and SSE** | There is no `Mcp-Session-Id` and no SSE stream: `GET` and `DELETE` answer `405` |
| **Notifications** (messages without an `id`) | `202` with no body |
| **Batches** (JSON-RPC array) | Accepted |
| **Protocol versions** | `2025-11-25`, `2025-06-18`, `2025-03-26`, `2024-11-05` (if the client asks for another one, it answers `2025-11-25`) |
| **Methods** | `initialize`, `ping`, `tools/list`, `tools/call`. `resources/list`, `resources/templates/list` and `prompts/list` return empty lists; any other method gets `-32601` |
| **`MCP-Protocol-Version` header** | Not required |

`initialize` returns `serverInfo` with the classroom name (`Didacta · <name>`) and `instructions` for the model (in English, quoting screen names in the classroom's language): what it can do, that new courses are drafts, and that it must write content in the language the owner uses.

!!! note "No CORS, like the rest of the API"
    MCP clients connect from your machine or from a server, not from a web page. See [Conventions](conventions.md#other-conventions).

## Authentication

The key goes in the `Authorization` header, with either of these two schemes:

```http
Authorization: Bearer lmsk_…
Authorization: ApiKey lmsk_…
```

`Bearer` is what every MCP client offers through its "token" option; `ApiKey` is the scheme used by the rest of Didacta's API. The key is not accepted in the query string or in any other header, and **a session JWT does not get in**: this gateway is only for keys of their own, revocable and scoped.

| HTTP status | `code` | When |
|---|---|---|
| 401 | `AGENT_KEY_MISSING` | The header is missing or the key does not start with `lmsk_` |
| 401 | `AGENT_KEY_INVALID` | The key does not exist, has expired or was revoked |
| 403 | `AGENT_KEY_NOT_AGENT` | The key is valid but has no `agent:*` scope |
| 403 | `AGENT_KEY_OTHER_ACADEMY` | The request arrives through another classroom's domain |

### One key, one classroom

The classroom comes **from the key and from nothing else**: no header, parameter or tool argument can change it (tools do not even have a classroom argument). The domain only serves to **reject**: pasting a classroom's key into another classroom's URL returns `AGENT_KEY_OTHER_ACADEMY` instead of writing where it should not. A `courseId` from another classroom simply does not exist for this one.

## Scopes

They are product verbs, not mirrors of endpoints:

| Scope | Grants |
|---|---|
| `agent:courses:read` | List courses and read a course with its lessons |
| `agent:courses:write` | Create courses and lessons; rename and describe courses; edit lessons |
| `agent:live:write` | See upcoming live classes and schedule a new one |
| `agent:community:post` | See the community spaces and post in one |

`tools/list` returns **only** the tools the key's scopes allow **and** whose module is active in the classroom (`mod.courses`, `mod.zoom-live`, `mod.community`): the agent never sees something it will later be denied. The other API scopes (`enrollments:*`, `courses:read`, `orders:*`, `community:post`) belong to other APIs and do not open this gateway; see [Selling and integrating from outside](external-integrations.md).

## Tools

There are 10. Arguments are always validated on the server; an unknown field is an error (`additionalProperties: false`). The descriptions the model reads are in English; what the agent **creates** is written in the language you ask it to use.

| Tool | Scope | Module | Writes |
|---|---|---|---|
| `list_courses` | `agent:courses:read` | `mod.courses` | No |
| `get_course` | `agent:courses:read` | `mod.courses` | No |
| `create_course` | `agent:courses:write` | `mod.courses` | Yes |
| `update_course` | `agent:courses:write` | `mod.courses` | Yes |
| `create_lesson` | `agent:courses:write` | `mod.courses` | Yes |
| `update_lesson` | `agent:courses:write` | `mod.courses` | Yes |
| `list_live_classes` | `agent:live:write` | `mod.zoom-live` | No |
| `schedule_live_class` | `agent:live:write` | `mod.zoom-live` | Yes |
| `list_community_spaces` | `agent:community:post` | `mod.community` | No |
| `create_community_post` | `agent:community:post` | `mod.community` | Yes |

Read-only tools are advertised with `readOnlyHint: true`; none carries `destructiveHint`, because none deletes.

### Courses and lessons

**`list_courses`** — Lists the classroom's courses with `id`, `title`, `status` (`DRAFT`, `PUBLISHED`, `ARCHIVED`), number of lessons and `editPath`. Returns at most 100.

| Parameter | Type | Notes |
|---|---|---|
| `status` | enum, optional | Only courses in that status |

**`get_course`** — One course with its lessons in order: `id`, `number`, `title`, `type` and `hasContent` (whether the lesson already has content).

| Parameter | Type | Notes |
|---|---|---|
| `courseId` | uuid, required | |

**`create_course`** — Creates a course **as a draft**. The web address comes from the title (if it already exists it tries `-2`, `-3`… up to 20 times); the course language is the one in the profile of whoever created the key.

| Parameter | Type | Notes |
|---|---|---|
| `title` | text 1–160, required | |
| `description` | text ≤ 2000, optional | Plain text |

**`update_course`** — Changes the title and/or the description. It does not touch the status or the price.

| Parameter | Type | Notes |
|---|---|---|
| `courseId` | uuid, required | |
| `title` | text 1–160 | At least one of `title` or `description` |
| `description` | text ≤ 2000 | |

**`create_lesson`** — Adds a lesson **at the end** of the course (in its last section; if the course has none, it creates one with the default name in the owner's language: "Lecciones", "Lessons", "Aulas"). Call it once per lesson, in order, and do not number the titles: the classroom numbers them itself.

| Parameter | Type | Notes |
|---|---|---|
| `courseId` | uuid, required | |
| `title` | text 1–160, required | |
| `type` | `text` \| `video`, required | |
| `text` | text ≤ 100,000 | Only with `type: "text"`. Shown as **plain text** (line breaks are kept; neither HTML nor Markdown is rendered) |
| `videoUrl` | `https://` URL ≤ 2000 | Only with `type: "video"` (YouTube, Vimeo, Bunny…) |

**`update_lesson`** — Changes the title, the text (text lessons) or the video link (video lessons). The type does not change.

| Parameter | Type | Notes |
|---|---|---|
| `lessonId` | uuid, required | |
| `title` | text 1–160 | At least one of `title`, `text` or `videoUrl` |
| `text` | text ≤ 100,000 | `AGENT_WRONG_TYPE` if the lesson is not a text lesson |
| `videoUrl` | `https://` URL | `AGENT_WRONG_TYPE` if the lesson is not a video lesson |

### Live classes

**`list_live_classes`** — The upcoming classes (cancelled ones excluded), ordered by date, up to 20: `id`, `title`, `startTime`, `timezone`, `durationMinutes`, `status` and `path`. No parameters.

**`schedule_live_class`** — Schedules a class that students join through **the video-call link you already use** (Zoom, Google Meet, Teams, Jitsi…). With your own link, this tool never talks to the Zoom API.

| Parameter | Type | Notes |
|---|---|---|
| `title` | text 1–200, required | |
| `startTime` | ISO 8601 date-time **with offset**, required | `2026-11-20T19:00:00+01:00` |
| `durationMinutes` | integer 15–480, required | |
| `timezone` | IANA zone, required | `Europe/Madrid` |
| `joinUrl` | `https://` URL, required | Video-call link |
| `description` | text ≤ 2000 | |
| `courseId` | uuid | Links the class to a course |
| `announce` | boolean | With `true`, it is also announced in the community |

The class is created with the key owner's email as host and is **visible straight away**.

### Community

**`list_community_spaces`** — The community spaces (`slug`, `title`). No parameters.

**`create_community_post`** — Publishes a post signed by the key's owner. It **does not email the members** and is not flagged as important.

| Parameter | Type | Notes |
|---|---|---|
| `title` | text 3–200, required | |
| `body` | text 1–10,000, required | |
| `space` | slug (`[a-z0-9-]`, ≤ 60) | If omitted, it goes to `general`. A space that does not exist gives `AGENT_SPACE_UNKNOWN` with the list of valid ones |

If the key owner has a community sanction that stops them from posting, their agent cannot either (`AGENT_OWNER_RESTRICTED`).

## Calling the gateway by hand

Handy for testing a key:

```bash
curl -X POST https://your-classroom.example.com/api/v1/mcp \
  -H "Authorization: Bearer lmsk_…" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc": "2.0", "id": 1, "method": "tools/list"}'
```

And a tool call:

```bash
curl -X POST https://your-classroom.example.com/api/v1/mcp \
  -H "Authorization: Bearer lmsk_…" \
  -H "Content-Type: application/json" \
  -d '{
        "jsonrpc": "2.0",
        "id": 2,
        "method": "tools/call",
        "params": {
          "name": "create_course",
          "arguments": { "title": "Introduction to photography", "description": "Four weeks to master your camera." }
        }
      }'
```

A tool's result arrives as a `content` block of type `text` holding the result JSON (for `create_course`: `id`, `title`, `status`, `editPath` and a `note`). Failures **of the tool** are not protocol errors: they come back as a result with `isError: true` and the text `message (CODE)`, so the agent can read it and fix the call or tell the person.

| `code` | Meaning |
|---|---|
| `AGENT_TOOL_UNKNOWN` | The tool does not exist |
| `AGENT_SCOPE_MISSING` | The key lacks the scope the tool needs |
| `AGENT_MODULE_OFF` | The tool's module is turned off in this classroom |
| `AGENT_RATE_LIMITED` | Call limit exceeded (see below) |
| `AGENT_INVALID_ARGUMENTS` | Arguments that do not match the schema |
| `AGENT_OWNER_GONE` / `AGENT_OWNER_NOT_ADMIN` | Whoever created the key is no longer active or no longer an administrator |
| `AGENT_NOT_FOUND`, `AGENT_WRONG_TYPE`, `AGENT_COURSE_SLUG`, `AGENT_SPACE_UNKNOWN`, `AGENT_OWNER_RESTRICTED` | Business failures of each tool (see above) |
| `AGENT_INTERNAL` | Unexpected error; internal details are not leaked |

Errors raised by the modules themselves (courses, live classes) keep their own code.

## Limits

- **Per key**: only `tools/call` requests count. By default **60 per minute** and **600 per hour**; tune them with `AGENT_TOOL_CALLS_PER_MIN` and `AGENT_TOOL_CALLS_PER_HOUR` (see [Environment variables](../configuration/environment-variables.md)). When exceeded, the tool returns `AGENT_RATE_LIMITED` with the seconds to wait. With Redis the counter is shared across replicas; without Redis it counts in the process's memory (it does not let everything through).
- **Per classroom**: the API's general request limit also applies, and all the classroom's keys share it.
- **Sizes**: see each tool's table. `list_courses` returns up to 100 courses and `list_live_classes` up to 20 classes.

## Configuring an MCP client

The panel generates these blocks already filled in with your classroom URL and your key; here they use example values. `classroom-name` is the name the client stores the connection under (the panel uses the classroom's identifier —its *slug*— in lowercase with hyphens).

=== "Claude Code"

    ```bash
    claude mcp add --transport http --scope user classroom-name \
      https://your-classroom.example.com/api/v1/mcp \
      --header "Authorization: Bearer lmsk_YOUR_KEY"

    claude mcp list   # to check it connected
    ```

=== "Codex"

    In `~/.codex/config.toml`:

    ```toml
    [mcp_servers.classroom-name]
    url = "https://your-classroom.example.com/api/v1/mcp"
    http_headers = { "Authorization" = "Bearer lmsk_YOUR_KEY" }
    ```

    Check: `codex mcp list`.

=== "Hermes"

    In `~/.hermes/config.yaml`:

    ```yaml
    mcp_servers:
      classroom-name:
        url: "https://your-classroom.example.com/api/v1/mcp"
        headers:
          Authorization: "Bearer lmsk_YOUR_KEY"
    ```

    Check: `hermes mcp test classroom-name`.

=== "Grok"

    With Grok Build, in the terminal:

    ```bash
    grok mcp add --transport http classroom-name \
      https://your-classroom.example.com/api/v1/mcp \
      --header "Authorization: Bearer lmsk_YOUR_KEY"
    ```

    In the Grok app: *Connect apps → Add a custom connector → Remote HTTPS*, with the server URL and the header `Authorization: Bearer lmsk_YOUR_KEY`.

=== "Cursor"

    In `~/.cursor/mcp.json` (and enable it in Cursor's MCP settings):

    ```json
    {
      "mcpServers": {
        "classroom-name": {
          "url": "https://your-classroom.example.com/api/v1/mcp",
          "headers": { "Authorization": "Bearer lmsk_YOUR_KEY" }
        }
      }
    }
    ```

=== "VS Code"

    In your project's `.mcp.json` (GitHub Copilot in agent mode):

    ```json
    {
      "mcpServers": {
        "classroom-name": {
          "type": "http",
          "url": "https://your-classroom.example.com/api/v1/mcp",
          "headers": { "Authorization": "Bearer lmsk_YOUR_KEY" }
        }
      }
    }
    ```

!!! note "Only clients that send a fixed header"
    The gateway authenticates with a key in the `Authorization` header. A client that only supports OAuth cannot connect (that is the case of ChatGPT).

Once connected, ask for something concrete: *"Create a course called "Introduction to photography" with 6 text lessons from these notes…"* or *"Schedule a live class next Tuesday at 19:00 (my time) with this video-call link…"*.

## Security and audit

- **Isolation between classrooms**: the classroom comes from the key's row; see [above](#one-key-one-classroom). Each call also runs inside that classroom's context, so the database's row-level security (RLS) acts as a second lock.
- **Least privilege**: scopes trim which tools exist for the key, and what no scope grants (deleting, prices, publishing for sale, inviting, emailing everyone) **does not exist** on this gateway.
- **One revocable key per agent**: create one per agent or per person and revoke it instantly in **Administration → API keys**. Only the token's hash is stored; the plaintext token cannot be recovered.
- **Audit**: every tool that writes leaves an entry in the audit log (**Administration → Audit log**) with the key's owner as actor, the action `agent.<tool>` (`agent.create_course`, `agent.schedule_live_class`…), the affected resource and, in `metadata`, the `apiKeyId` and `via: "mcp"`. Read-only tools write no entries. Creating the key is recorded as `admin.api_key.created` (with `purpose: "agent"` when it came from the panel).
- **No special licence**: the agent gateway is not tied to any licence capability (none of its routes answers `402`), so it is available in every edition.

## See also

- [Authentication](authentication.md) — API keys, scopes and sessions.
- [Selling and integrating from outside](external-integrations.md) — the API-key APIs for shops and CRMs.
- [Courses and catalog](../modules/catalog/courses.md), [Zoom virtual classroom](../modules/catalog/zoom-live.md) and [Community](../modules/catalog/community.md) — the modules behind each tool.
