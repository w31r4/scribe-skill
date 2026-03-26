# Data Model

Understanding scribe's data model helps you ask the right questions and interpret
the output correctly.

## Entity hierarchy

```
Session
  └── Turn (1:N)
        └── TraceRequest (1:N)
              └── RawPayload (1:1)
```

## Session

One continuous agent interaction — e.g., one `claude` CLI run, one Codex sandbox,
one Cursor conversation.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | 32-char unique ID (UUIDv7-like) |
| `source` | string | Which tool: `claude-code`, `codex`, `cursor` |
| `started_at` | ISO timestamp | When the session began |
| `ended_at` | ISO timestamp | When it ended (null if active) |
| `metadata` | dict | Provider-specific metadata |

Sessions are created automatically when scribe detects a new conversation.
A session ends when there's a 5-minute gap in activity or the agent exits.

## Turn

One user↔agent exchange within a session. A turn starts when the user sends a
message and ends when the agent finishes responding (including all tool use loops).

| Field | Type | Description |
|-------|------|-------------|
| `turn_number` | int | Sequential within the session (0-indexed) |
| `status` | string | `active`, `completed`, `aborted` |
| `started_at` | ISO timestamp | When the turn began |
| `ended_at` | ISO timestamp | When it ended |
| `abort_reason` | string | Why it was aborted (if applicable) |

A single turn may contain many trace requests — the primary exchange plus
subagent spawns, tool use loops, assistive checks, and probes.

## TraceRequest

One HTTP request or response captured by the proxy. This is the core unit of
trace data.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | 32-char unique ID (the `trace_request_id`) |
| `session_id` | string | Parent session |
| `turn_id` | string | Parent turn |
| `request_id` | string | HTTP-level ID that pairs request with response |
| `direction` | string | `request` (what agent sent) or `response` (what came back) |
| `provider` | string | `anthropic`, `openai`, `codex`, `cursor` |
| `model` | string | Model name from the request |
| `parent_request_id` | string | Links subagent to parent (null for root requests) |
| `timestamp` | ISO timestamp | When captured |
| `summary` | dict | Extracted metadata (JSON): model, role, stop_reason, etc. |

The `request_id` field is key for pairing: one HTTP exchange produces two
trace_request rows with the same `request_id` but different `direction`.

## RawPayload

The exact HTTP body bytes, stored alongside the trace_request.

| Field | Type | Description |
|-------|------|-------------|
| `trace_request_id` | string | Links to the parent TraceRequest |
| `body` | bytes | Raw HTTP body |
| `content_type` | string | Usually `application/json` |

For JSON providers, body is the raw JSON bytes. For SSE responses, body is the
full SSE text. For Cursor, body may be protobuf bytes.

## Execution roles

Stored in `summary.request_role`, normalized by the query layer.

| Role | Detection method | Purpose |
|------|-----------------|---------|
| `primary` | Default; main conversation system prompt | The user↔agent exchange |
| `subagent` | Has `parent_request_id`, specific system prompt | Spawned sub-tasks (Explore, Plan, etc.) |
| `assistive` | Haiku model, specific system prompts | Lightweight helpers (hooks, topic detection) |
| `probe` | `max_tokens=1` or `/count_tokens` path | Token counting, capability checks |

## Fine roles

Stored in `summary.fine_role`, gives more detail than execution role.

### Primary
- `main_primary` — the main user-facing conversation

### Subagent
- `task_subagent` — general-purpose subagent
- `task_subagent_explore` — Explore agent (file search, codebase navigation)
- `task_subagent_plan` — Plan agent (architecture, implementation strategy)
- `task_subagent_guide` — Guide agent (documentation lookup)
- `task_subagent_statusline` — Statusline configuration agent

### Assistive
- `hook_assistive` — pre/post-tool hook (e.g., "is this safe to execute?")
- `topic_assistive` — topic change detection ("is this a new conversation?")
- `suggest_assistive` — next-action suggestions for the user
- `filepath_assistive` — file path extraction from shell commands
- `unknown_assistive` — unclassified helper (haiku + 0 tools)

### Probe
- `probe` — token counting or model capability check

## Provider detection

Scribe detects the provider from the HTTP request path:

| Path pattern | Provider |
|-------------|----------|
| `/v1/messages` | `anthropic` |
| `/v1/chat/completions` | `openai` |
| `/backend-api/codex/` | `codex` |
| gRPC/ConnectRPC | `cursor` |

Claude Code uses the Anthropic API. Codex uses both OpenAI-format endpoints
and its own first-party endpoints. Cursor uses ConnectRPC over HTTP/2.

## Storage location

All data lives in `~/.scribe/traces.db` (SQLite). Tables:

| Table | Contents |
|-------|----------|
| `sessions` | Session metadata |
| `turns` | Turn lifecycle |
| `trace_requests` | Request/response metadata + summary JSON |
| `raw_payloads` | Raw HTTP bodies |
| `cursor_stream_snapshots` | Cursor-specific stream state |

The database has read-optimized indexes on session_id, turn_id, timestamp,
and request_id for efficient querying.
