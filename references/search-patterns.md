# Searching Payloads

`scribe q search` decodes raw payloads on the fly and applies regex matching
against extracted message content. It's the primary way to find specific content
across sessions without knowing which request contains it.

## Basic syntax

```bash
scribe q search "<regex_pattern>" [options]
```

The pattern is a Python regex, always case-insensitive. Special regex characters
need escaping:

```bash
scribe q search "system-reminder"           # literal string
scribe q search "CLAUDE\.md"                # escaped dot
scribe q search "memory/[\w./-]+\.md"       # regex for memory file paths
scribe q search "tool_(use|result)"         # alternation
scribe q search 'explore|file search'       # regex OR (use single quotes!)
```

**Shell quoting matters for regex OR (`|`).** The pipe character is a shell
operator, so always wrap patterns containing `|` in **single quotes**:
- `scribe q search 'foo|bar'` — correct
- `scribe q search "foo|bar"` — correct
- `scribe q search foo\|bar` — may fail depending on shell

## Filtering by role

Messages have one of 4 roles. Filtering by role dramatically narrows results
and speeds up the search (fewer messages to check per payload).

```bash
# System prompts only
scribe q search "You are Claude" --role system

# User queries only
scribe q search "help me" --role user

# Model responses only
scribe q search "I'll help" --role assistant

# Tool call results only
scribe q search "error" --role tool
```

| Role | What it contains |
|------|-----------------|
| `system` | System prompt, CLAUDE.md, memory, context sections |
| `user` | Human messages, tool results (in some providers) |
| `assistant` | Model responses, tool use requests |
| `tool` | Tool execution results |

## Filtering by execution role

Use `--execution-role` and `--fine-role` to narrow search to specific request
types. This filters at the trace_request level (before message extraction),
which is more efficient than post-filtering with `--jq`.

```bash
# Only search in subagent requests
scribe q search "You are" --execution-role subagent --role system

# Only search in explore agents
scribe q search "file" --fine-role task_subagent_explore

# Only search in primary conversation
scribe q search "implement" --execution-role primary --role user
```

| Execution Role | What it is |
|---------------|-----------|
| `primary` | Main agent conversation |
| `subagent` | Spawned sub-task (Explore, Plan, General) |
| `assistive` | Lightweight helper (hooks, suggestions) |
| `probe` | Token counting / capability checks |

Search results include `execution_role` and `fine_role` (when available) in
each match, so you can see the request context without needing to cross-check
the tree.

## Filtering by direction

```bash
# Only what the agent sent to the API
scribe q search "temperature" --direction request

# Only what came back
scribe q search "stop_reason" --direction response
```

Combining role + direction is powerful:
```bash
# Find system-reminder injections in requests
scribe q search "<system-reminder>" --role system --direction request

# Find error responses
scribe q search "error" --role assistant --direction response
```

## Scoping to sessions

```bash
# Search only the latest session
scribe q search "pattern" --latest --source claude-code

# Search a specific session
scribe q search "pattern" --session 019d2463d141

# Search all sessions for a source (default: up to 50 sessions)
scribe q search "pattern" --source codex
```

Without `--latest` or `--session`, search scans the 50 most recent sessions.

## Controlling output

```bash
# More context around matches (default: 100 chars)
scribe q search "MEMORY.md" --context-chars 300

# No snippet truncation
scribe q search "MEMORY.md" --full

# More matches (default: 20)
scribe q search "tool_use" --limit 50
```

## Understanding the output

```json
{
  "pattern": "MEMORY.md",
  "total_matches": 3,
  "matches": [
    {
      "trace_request_id": "019d2463d146",
      "trace_request_id_full": "019d2463d1467418a721788299ac2cea",
      "session_id_short": "019d2463d141",
      "direction": "request",
      "role": "system",
      "match_field": "content",
      "snippet": "...add a pointer to that file in `MEMORY.md`. `MEMORY.md` is an index..."
    }
  ]
}
```

- `trace_request_id` — 12-char short ID, usable directly in `show`, `raw`, `diff`
- `trace_request_id_full` — full 32-char ID for exact matching
- `snippet` — context around the match, controlled by `--context-chars`

When `_search_truncated: true` appears, the search hit the 500-payload decode
limit. Narrow your scope with `--latest`, `--session`, or `--direction`.

## Chaining with other commands

```bash
# Find the request, then inspect it
scribe q search "MEMORY.md" --role system --latest \
  --jq '.matches[0].trace_request_id_full'
# Copy the ID, then:
scribe q show <id> --fields messages --full

# Or in one pipeline (requires jq installed)
ID=$(scribe q search "MEMORY.md" --role system --latest \
  --jq '.matches[0].trace_request_id_full' | tr -d '"')
scribe q show "$ID" --fields messages --full
```

## Performance notes

Search decodes every raw payload to extract messages — this is intentionally
brute-force (no search index). For large datasets:

1. Always scope with `--latest` or `--session` when possible
2. Use `--direction` to halve the payloads to decode
3. Use `--role` to skip non-matching messages early
4. Use `--limit` to stop after enough matches
