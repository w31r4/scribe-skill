# Tracking Prompt Evolution with Diff

`scribe q diff` finds the previous same-role request in a session and computes
what changed. This is the primary tool for understanding how prompts, tools, and
configuration evolve across turns.

## Basic usage

```bash
# Full diff (all sections)
scribe q diff 019578a1b2c3

# Specific sections only
scribe q diff <id> --fields system
scribe q diff <id> --fields tools
scribe q diff <id> --fields memory
scribe q diff <id> --fields config
scribe q diff <id> --fields messages
scribe q diff <id> --fields usage

# Multiple sections
scribe q diff <id> --fields system,tools,memory

# No truncation
scribe q diff <id> --full
```

The `--fields` flag both selects which sections to compute and which to output.
This matters for performance — omitting sections you don't need skips the
extraction work.

## How diff finds the "previous" request

Diff works within a single session. It:

1. Loads all `direction=request` trace entries for the session
2. Filters to the same `execution_role` as the target (e.g., both `primary`)
3. Walks the list chronologically and returns the one immediately before the target

This means diff compares a primary request against the previous primary request,
a subagent against the previous subagent, etc. Cross-role comparison is not
supported — it would not be meaningful since different roles have very different
payload structures.

## Baseline vs delta

The first request of its role in a session has no predecessor. Diff returns:

```json
{
  "trace_request_id": "...",
  "compared_to": null,
  "status": "baseline",
  "messages": [...],
  "tool_definitions_count": 54,
  "usage": {}
}
```

Subsequent requests return `"status": "delta"` with change details.

## Section: system_prompt

Tracks changes to the system prompt (all `role=system` messages concatenated).

```bash
scribe q diff <id> --fields system
```

When unchanged:
```json
{"system_prompt": {"changed": false, "chars": 27823}}
```

When changed:
```json
{
  "system_prompt": {
    "changed": true,
    "prev_chars": 27823,
    "current_chars": 28156,
    "diff_lines": 12,
    "diff": "--- \n+++ \n@@ -150,3 +150,6 @@\n+new content here..."
  }
}
```

The `diff` field is a unified diff. Use `--full` to see it untruncated.

## Section: tools

Shows which tool definitions were added or removed.

```bash
scribe q diff <id> --fields tools
```

```json
{
  "tools": {
    "added": ["NewTool"],
    "removed": ["OldTool"],
    "unchanged_count": 52
  }
}
```

This compares tool names only, not schemas. To see schema details, use
`scribe q show <id> --fields tools`.

## Section: memory

Tracks memory file paths mentioned in system messages.

```bash
scribe q diff <id> --fields memory
```

```json
{
  "memory": {
    "added_paths": ["memory/feedback_testing.md"],
    "removed_paths": [],
    "unchanged_count": 5
  }
}
```

Matches paths like `memory/user_profile.md`, `memory/project_goals.md`, etc.
from system message content using regex `memory/[\w./-]+\.md`.

## Section: config

Compares top-level API config parameters.

```bash
scribe q diff <id> --fields config
```

```json
{
  "config": {
    "model": {"from": "claude-sonnet-4-6", "to": "claude-opus-4-6"},
    "max_tokens": {"from": 4096, "to": 8192}
  }
}
```

Or when nothing changed:
```json
{"config": {"changed": false}}
```

Tracked keys: `model`, `max_tokens`, `temperature`, `top_p`, `top_k`.

## Section: messages

Shows new messages added since the previous request — typically the user's query
and any tool results from the last turn.

```bash
scribe q diff <id> --fields messages --full
```

```json
{
  "new_messages": [
    {"role": "assistant", "content": "I'll help you with that..."},
    {"role": "user", "content": "Now implement the tests"}
  ],
  "total_messages": {"previous": 4, "current": 8}
}
```

The `total_messages` field shows the conversation growing across turns.

## Section: usage

Token count comparison.

```bash
scribe q diff <id> --fields usage
```

```json
{
  "usage": {
    "current": {"input_tokens": 15234, "output_tokens": 0},
    "previous": {"input_tokens": 12100, "output_tokens": 0},
    "delta": {"input_tokens": 3134, "output_tokens": 0}
  }
}
```

Note: request-direction usage reflects input tokens (what the agent sent).
Response-direction usage is on the paired response. Diff only works on
request-direction entries.

## Common workflows

### Track system prompt growth over a session

```bash
# 1. Get all primary request IDs
scribe q tree --latest --source claude-code \
  --jq '[.turns[].requests[] | select(.execution_role=="primary") | .trace_request_id]'

# 2. Diff each one (skip the first — it's a baseline)
scribe q diff <turn2_id> --fields system
scribe q diff <turn3_id> --fields system
```

### Find when a tool was added/removed

```bash
# Diff each turn and look for tool changes
scribe q diff <id> --fields tools --jq '.tools | select(.added != [] or .removed != [])'
```

### Monitor memory loading

```bash
scribe q diff <id> --fields memory \
  --jq '.memory | select(.added_paths != [])'
```

## Best practices

1. **Start with `tree`** to find the trace_request_ids, then use `diff` on them.
   Don't guess IDs.

2. **Use `--fields`** to focus on what you care about. Extracting all sections
   is slower than extracting one.

3. **Use `--full`** for system prompt diffs. The default truncation (500 chars)
   often cuts off useful context in unified diffs.
