# Understanding the Request Tree

`scribe q tree` visualizes the hierarchical structure of a session: which requests
are primary, which are subagents, who spawned whom, and what the outcome was.

## Basic usage

```bash
# Latest session, human-readable
scribe q tree --latest --source claude-code --table

# JSON output for programmatic use
scribe q tree --latest --source claude-code

# Specific session
scribe q tree --session 019d2463d141

# Single turn only
scribe q tree --latest --source claude-code --turn 3
```

## Reading the table output

```
Session: 019d2463d141  Source: claude-code

Turn 1
  019d2463d4cd  claude-haiku-4-5-20251001  -> end_turn
  019d2463d146  claude-opus-4-6  (main_primary)

Turn 2
  019d2463d607  claude-opus-4-6  (main_primary) -> end_turn
  019d2463f347  claude-haiku-4-5-20251001 [assistive] (hook_assistive) -> tool_use

Turn 3
  019d24661c1f  claude-opus-4-6  (main_primary) -> tool_use
    019d24663909  claude-haiku-4-5-20251001 [subagent] (task_subagent_explore) -> tool_use
      019d2466444e  claude-haiku-4-5-20251001 [subagent] (task_subagent_explore) -> tool_use
  019d24665bbe  claude-opus-4-6 [probe] (probe)
  019d24679ae0  claude-opus-4-6  (main_primary) -> end_turn
```

Each line shows:
- **Indentation** — parent-child nesting depth
- **12-char ID** — the trace_request_id (usable in `show`, `raw`, `diff`)
- **Model name** — which LLM handled this request
- **[role]** — execution role if not primary (subagent, assistive, probe)
- **(fine_role)** — detailed classification (main_primary, task_subagent_explore, etc.)
- **-> stop_reason** — how the request ended (end_turn, tool_use, etc.)

## Execution roles

| Role | What it is | Examples |
|------|-----------|----------|
| `primary` | The main agent conversation | User ↔ Claude interaction |
| `subagent` | A spawned sub-task | Explore agent, Plan agent, General agent |
| `assistive` | Lightweight helper | Hook checks, topic detection, suggestions |
| `probe` | Capability/token check | `max_tokens=1` requests, `/count_tokens` |

## Fine roles

Fine roles give more detail about what a request is doing. They appear in
parentheses in `--table` output.

**Primary:**
- `main_primary` — the main user-facing conversation

**Subagent:**
- `task_subagent` — general-purpose subagent (legacy name)
- `task_subagent_general` — general-purpose subagent (current, all tools)
- `task_subagent_explore` — Explore agent (file search, READ-ONLY)
- `task_subagent_plan` — Plan agent (architecture, implementation planning)
- `task_subagent_guide` — Guide agent (documentation lookup)

**Assistive:**
- `hook_assistive` — pre/post-tool hook check (e.g., "is this safe to execute?")
- `topic_assistive` — topic change detection
- `suggest_assistive` — next-action suggestions
- `filepath_assistive` — file path extraction from commands
- `websearch_assistive` — web search helper
- `unknown_assistive` — haiku model with 0 tools (unclassified helper)

**Probe:**
- `probe` — token counting or model capability probes

## Parent-child relationships

Subagent requests have a `parent_request_id` linking them to the primary request
that spawned them. The tree renders this as indentation:

```
  019d24661c1f  claude-opus-4-6  (main_primary) -> tool_use      ← parent
    019d24663909  claude-haiku-4-5-20251001 [subagent] -> tool_use  ← child (round 1)
      019d2466444e  claude-haiku-4-5-20251001 [subagent] -> tool_use  ← grandchild (round 2)
```

Each subagent round chains as child-of-child because the previous round's
response is the parent of the next round's request.

## JSON output structure

```json
{
  "session_id_short": "019d2463d141",
  "session_id": "019d2463d1417418a721788299ac2cea",
  "source": "claude-code",
  "turns": [
    {
      "turn_number": 1,
      "turn_id": "...",
      "status": "completed",
      "started_at": "2026-03-25T09:46:54Z",
      "ended_at": "2026-03-25T09:47:10Z",
      "requests": [
        {
          "trace_request_id": "019d2463d146...",
          "request_id": "req_abc123",
          "model": "claude-opus-4-6",
          "provider": "anthropic",
          "execution_role": "primary",
          "fine_role": "main_primary",
          "timestamp": "...",
          "stop_reason": "end_turn",
          "response_trace_request_id": "019d2463d4cd...",
          "children": [
            {
              "trace_request_id": "019d24663909...",
              "execution_role": "subagent",
              "fine_role": "task_subagent_explore",
              "children": [...]
            }
          ]
        }
      ]
    }
  ]
}
```

## Filtering by role

Use `--role` and `--fine-role` to narrow the tree to specific request types:

```bash
# Only subagent requests (preserves parent hierarchy for context)
scribe q tree --latest --source claude-code --role subagent --table

# Only explore agents
scribe q tree --latest --source claude-code --fine-role task_subagent_explore --table

# Only hook assistive requests
scribe q tree --latest --fine-role hook_assistive --table
```

The filter preserves parent nodes when their children match, so you still see
the hierarchy context. Turns with no matching requests are omitted entirely.

## Common queries with jq

```bash
# Count total requests per turn
scribe q tree --latest --source claude-code \
  --jq '.turns[] | {turn: .turn_number, count: (.requests | length)}'

# Find all subagent trace IDs (or use --role subagent)
scribe q tree --latest --source claude-code \
  --jq '[.turns[].requests[] | recurse(.children[]?) | select(.execution_role=="subagent") | .trace_request_id]'

# Find turns with aborted status
scribe q tree --latest --source claude-code \
  --jq '.turns[] | select(.status=="aborted")'

# Count subagent rounds per primary request
scribe q tree --latest --source claude-code \
  --jq '.turns[].requests[] | select(.children | length > 0) | {id: .trace_request_id, subagents: (.children | length)}'
```

## Turn statuses

| Status | Meaning |
|--------|---------|
| `active` | Turn is still in progress |
| `completed` | Turn finished normally |
| `aborted` | Turn was cancelled or errored out |
