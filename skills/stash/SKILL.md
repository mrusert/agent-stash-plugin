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

### Load Content

```bash
curl -s "https://agentstash.ai/stash/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns raw `text/plain`. Reading extends the TTL automatically.

### List Stashes

```bash
curl -s "https://agentstash.ai/stashes" \
  -H "X-API-Key: $STASH_API_KEY"
```

### Delete

```bash
curl -s -X DELETE "https://agentstash.ai/stash/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

### Append

```bash
curl -s -X PATCH "https://agentstash.ai/stash/{name}/append" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: text/plain" \
  -d '<content to append>'
```

### Find and Replace

```bash
curl -s -X PATCH "https://agentstash.ai/stash/{name}/replace" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"old": "<find>", "new": "<replace>"}'
```

Add `?all=true` to replace all occurrences.

## Channels (Multi-Agent Coordination)

Append-only streams for coordinating between agents.

### Write to Channel

```bash
curl -s -X POST "https://agentstash.ai/channel/{name}?create_if_missing=true" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "<message>", "label": "<optional-label>"}'
```

### Read Channel

```bash
curl -s "https://agentstash.ai/channel/{name}" \
  -H "X-API-Key: $STASH_API_KEY"

# Filter by label
curl -s "https://agentstash.ai/channel/{name}?label=<label>" \
  -H "X-API-Key: $STASH_API_KEY"
```

## Keeps (Persistent Storage)

Keeps are persistent named text files that never expire. Use them for data that
should survive indefinitely — agent personality, configuration, templates,
reference material. Available on all tiers.

### Create or Overwrite a Keep

```bash
curl -s -X PUT "https://agentstash.ai/keep/{name}" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: text/plain" \
  -d '<content>'
```

- `{name}`: alphanumeric, hyphens, dots, underscores (max 128 chars)

### Read a Keep

```bash
curl -s "https://agentstash.ai/keep/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

Returns raw `text/plain`.

### Append to a Keep

```bash
curl -s -X PATCH "https://agentstash.ai/keep/{name}/append" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: text/plain" \
  -d '<content to append>'
```

### Find and Replace in a Keep

```bash
curl -s -X PATCH "https://agentstash.ai/keep/{name}/replace" \
  -H "X-API-Key: $STASH_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"old": "<find>", "new": "<replace>"}'
```

Add `?all=true` to replace all occurrences.

### Delete a Keep

```bash
curl -s -X DELETE "https://agentstash.ai/keep/{name}" \
  -H "X-API-Key: $STASH_API_KEY"
```

### List All Keeps

```bash
curl -s "https://agentstash.ai/keeps" \
  -H "X-API-Key: $STASH_API_KEY"
```

## Tier Limits

| Tier | Max Stashes | Stash Max Size | Stash TTL | Max Keeps | Keep Max Size | Rate Limit |
|------|-------------|----------------|-----------|-----------|---------------|------------|
| Free | 5 | 64 KB | 24 hours | 1 | 1 MB | 10 req/min |
| Pro | 25 | 256 KB | 7 days | 10 | 5 MB | 60 req/min |
| Enterprise | 100 | 1 MB | 30 days | 50 | 25 MB | 300 req/min |

## Error Handling

| Status | Meaning | What to Do |
|--------|---------|------------|
| 404 | Not found | Stash/keep may have expired (stashes only) or name is wrong |
| 402 | Payment required | Free tier limit hit — tell user about upgrade options |
| 413 | Too large | Content exceeds tier size limit — split it up |
| 429 | Rate limited | Wait and retry after a few seconds |

## Usage Guidelines

- **Use keeps for persistent data**: Agent personality, configuration, templates,
  and reference material belong in keeps — they never expire and survive
  indefinitely
- **Use stashes for ephemeral working memory**: Intermediate results, session
  state, handoff context, and anything temporary belongs in stashes — they
  auto-expire based on your tier's TTL
- **Name stashes and keeps descriptively**: Use names like `project-state`,
  `research-notes`, `agent-personality` — not `temp1` or `data`
- **Stash state at natural breakpoints**: End of a task, before context gets too
  large, when switching focus
- **Content is plain text**: Markdown, JSON, YAML, code — anything that
  serializes to text works great
- **Keep content focused**: One stash/keep per concern. A stash called
  `api-research` should contain API research, not also your TODO list
