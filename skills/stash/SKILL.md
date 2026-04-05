---
name: stash
description: >
  Store and retrieve working memory using Agent Stash. Use when the user wants
  to save state for a future session, share context between agents, stash
  intermediate results, or resume previous work. Also use when context is
  getting large and offloading to external storage would help.
argument-hint: "[save|load|list|delete] [name]"
---

# Agent Stash — Persistent Working Memory for Claude Code

You have access to Agent Stash, a cloud API for storing and retrieving named
text files that persist across sessions. Use it to save working state, share
context between agents, or offload large intermediate results.

## Authentication

**Check for `$STASH_API_KEY` first.** If it is set, use it as the API key for
all requests (via `X-API-Key` header) and skip this section entirely.

**If `$STASH_API_KEY` is not set**, walk the user through setup:

1. Ask: "Do you already have an Agent Stash API key, or would you like me to
   register a new agent for you? (I can auto-register for free)"

2. **If they have a key**: Ask them to provide it. Use it for this session.

3. **If they want to register**: Ask "What would you like to name your agent?
   (default: `<current project directory name>`)" — use `basename` of the
   working directory as the suggested default. If they say "sure", "default",
   "whatever", or similar, use the default name.

4. **To register**, run:
   ```bash
   curl -s -X POST "https://agentstash.ai/register/agent" \
     -H "Content-Type: application/json" \
     -d '{"agent_name": "<chosen_name>"}'
   ```
   This returns `{"api_key": "sk_...", "agent_name": "..."}`.

5. **After obtaining the key** (either provided or registered), tell the user:
   "Your API key is `<key>`. To make it permanent, add this to your shell
   profile (`~/.zshrc` or `~/.bashrc`):
   ```
   export STASH_API_KEY="<key>"
   ```
   I'll use it for the rest of this session."

6. Use the key for all subsequent requests in this session.

## Base URL

`https://agentstash.ai`

## Operations

All requests include `-H "X-API-Key: $STASH_API_KEY"`.

### Save Content

```bash
curl -s -X PUT "https://agentstash.ai/stash/{name}" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: text/plain" \
  -d '<content>'
```

- `{name}`: alphanumeric, hyphens, dots, underscores (max 128 chars)
- Optional: `?ttl=<seconds>` to set expiration (default: tier-based)
- Optional: `?shared=true` to make readable by anyone who knows the name
- Optional: `?persistent=true` to store permanently (no TTL, survives restarts)

### Load Content

```bash
curl -s "https://agentstash.ai/stash/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns raw `text/plain`. Reading extends the TTL automatically (ephemeral only).

Add `?persistent=true` to read from persistent storage.

### List Stashes

```bash
curl -s "https://agentstash.ai/stashes" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns both ephemeral and persistent stashes, with a `persistent` flag on each.

### Delete

```bash
curl -s -X DELETE "https://agentstash.ai/stash/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

Add `?persistent=true` to delete from persistent storage.

### Append

```bash
curl -s -X PATCH "https://agentstash.ai/stash/{name}/append" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: text/plain" \
  -d '<content to append>'
```

Add `?persistent=true` to append to persistent storage.

### Find and Replace

```bash
curl -s -X PATCH "https://agentstash.ai/stash/{name}/replace" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"old": "<find>", "new": "<replace>"}'
```

Add `?all=true` to replace all occurrences. Add `?persistent=true` for persistent stashes.

## Streams (Multi-Agent Coordination)

Append-only logs for coordinating between agents. Each stream has an
unguessable `stream_id` — anyone with the ID can read and write.

### Create Stream

```bash
curl -s -X POST "https://agentstash.ai/stream" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "<stream-name>", "ttl": 3600}'
```

Returns `{"stream_id": "st_...", "name": "...", ...}`. Share `stream_id` with
collaborating agents.

Optional fields in the JSON body:
- `create_if_missing` (bool): If a stream with the same name already exists
  for this user, return it instead of creating a new one. Useful for
  idempotent initialization.
- `discoverable` (bool): Make the stream findable by name via
  `GET /streams?name=...`. Default `false`.
- `renew_on_access` (bool): Reset the TTL every time the stream is read or
  written. Keeps active streams alive. Default `false`.
- `overflow` (`"reject"` or `"prune"`): What happens when the stream hits
  its max entries. `"reject"` (default) returns 409; `"prune"` auto-deletes
  the oldest entries to make room.
- `retention_seconds` (int or null): Auto-expire individual entries after
  this many seconds. `null` (default) means entries live as long as the
  stream does. Useful for rolling-window data feeds.

### Write to Stream

```bash
curl -s -X POST "https://agentstash.ai/stream/{stream_id}" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"data": "<message>", "label": "<optional-label>"}'
```

Use labels to categorize entries (e.g., "research", "analysis", "status").

### Read Stream

```bash
curl -s "https://agentstash.ai/stream/{stream_id}" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns entries in chronological order (oldest first).

Optional query parameters:
- `?label=<label>` — Filter entries by label
- `?limit=<n>` — Max entries per page (1-1000, default 100)
- `?offset=<n>` — Skip entries for pagination

### Read Latest Entry

```bash
curl -s "https://agentstash.ai/stream/{stream_id}/latest" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns only the most recent entry. Add `?label=<label>` to get the latest
entry with a specific label.

### Extend Stream TTL

```bash
curl -s -X PATCH "https://agentstash.ai/stream/{stream_id}" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"extra_time": 1800}'
```

Adds seconds to the stream's remaining TTL (capped by your tier's max TTL).

### Look Up Stream by Name

```bash
curl -s "https://agentstash.ai/streams?name=<stream-name>" \
  -H "X-API-Key: $STASH_API_KEY"
```

Finds a discoverable stream by name (case-insensitive). Returns its
`stream_id` so you can read/write to it.

### Discover Resources

```bash
curl -s "https://agentstash.ai/discover?type=stream&label=<label>"
```

No auth required. Query parameters:
- `type` — `stash` or `stream`
- `label` — Filter by label
- `community` — `true` to show only community resources
- `limit` — Max results (1-100, default 50)
- `offset` — Pagination offset

### Community Streams

Community streams are public, curated data feeds:
- **Read**: Free, no authentication required
- **Write**: Requires x402 micropayment (`X-Payment` header)
- **Create**: Only the system account can create them

To read a community stream without auth:

```bash
curl -s "https://agentstash.ai/stream/{stream_id}"
```

## Persistent Stashes

Add `?persistent=true` to any stash endpoint to store data permanently in
PostgreSQL. Persistent stashes never expire and survive restarts. Use them for
agent personality, configuration, templates, and reference material.

All the same operations work (PUT, GET, PATCH, DELETE) — just add the query
parameter. Persistent stashes have separate, larger size limits.

## Tier Limits

| Tier | Stashes | Persistent | Stash Size | Persistent Size | Max TTL | Streams | Entries/Stream | Rate Limit |
|------|---------|------------|------------|-----------------|---------|---------|----------------|------------|
| Free | 5 | 1 | 64 KB | 1 MB | 24h | 5 | 50 | 10 req/min |
| Pro | 25 | 10 | 256 KB | 5 MB | 7d | 50 | 500 | 60 req/min |
| Enterprise | 100 | 50 | 1 MB | 25 MB | 30d | 500 | 5,000 | 300 req/min |
| Agent (x402) | 25 | 0 | 256 KB | N/A | 7d | 50 | 500 | 120 req/min |

**Note:** The Agent tier is activated by attaching an `X-Payment` header
(x402 micropayment). Free-tier users can attach `X-Payment` alongside their
`X-API-KEY` to temporarily elevate to Agent limits per-request. Agent tier
cannot use persistent stashes.

## Error Handling

| Status | Meaning | What to Do |
|--------|---------|------------|
| 404 | Not found | Stash may have expired (ephemeral only) or name is wrong |
| 402 | Payment required | Free tier limit hit — tell user about upgrade options or x402 payment |
| 403 | Forbidden | Persistent stashes not available on Agent/x402 tier |
| 413 | Too large | Content exceeds tier size limit — split it up |
| 429 | Rate limited | Wait and retry after a few seconds |

## Response Headers

When reading stashes, the API returns metadata headers:
- `X-Stash-Persistent`: `true` or `false`
- `X-Stash-Expires-At`: ISO timestamp (empty for persistent stashes)
- `X-Stash-Size`: Content size in bytes

## Usage Guidelines

- **Use persistent stashes for permanent data**: Agent personality, configuration,
  templates, and reference material belong in persistent stashes (`?persistent=true`)
  — they never expire
- **Use stashes for ephemeral working memory**: Intermediate results, session
  state, handoff context, and anything temporary belongs in regular stashes —
  they auto-expire based on your tier's TTL
- **Use streams for multi-agent coordination**: When multiple agents need to
  share an ordered log of data, use streams with labels for organization
- **Name stashes descriptively**: Use names like `project-state`,
  `research-notes`, `agent-personality` — not `temp1` or `data`
- **Stash state at natural breakpoints**: End of a task, before context gets too
  large, when switching focus
- **Content is plain text**: Markdown, JSON, YAML, code — anything that
  serializes to text works great
- **Keep content focused**: One stash per concern. A stash called
  `api-research` should contain API research, not also your TODO list
