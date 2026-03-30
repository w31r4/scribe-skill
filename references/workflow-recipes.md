# Multi-Command Workflow Recipes

These recipes combine multiple `scribe q` commands for common analysis scenarios.
Each starts with the user's goal and shows the exact command sequence.

## Reverse-engineer what an agent sees

**Goal**: See the complete context a Claude Code agent receives in a specific turn.

```bash
# 1. Find the primary request for the turn you care about
scribe q tree --latest --source claude-code --table
# Pick the primary request ID from the turn

# 2. Show all messages (the full conversation context)
scribe q show <primary_id> --fields messages --full

# 3. Show tool definitions (what tools are available)
scribe q show <primary_id> --fields tools --full | jq '.tool_definitions[].name'

# 4. See the raw system prompt structure
scribe q raw <primary_id> --jq '.system'
```

## Compare system prompts across providers

**Goal**: See how Claude Code vs Codex vs Cursor frame their system instructions.

```bash
# Find system prompt content for each source
scribe q search "You are" --role system --source claude-code --latest --limit 1
scribe q search "You are" --role system --source codex --latest --limit 1
scribe q search "You are" --role system --source cursor --latest --limit 1

# Get full system prompts
scribe q show <claude_id> --fields messages --full \
  | jq '[.messages[] | select(.role=="system")] | length'
scribe q show <codex_id> --fields messages --full \
  | jq '[.messages[] | select(.role=="system")] | length'
```

## Audit subagent patterns

**Goal**: Understand how a session used subagents — how many, what types, what
they accomplished.

```bash
# 1. Get only subagent requests in the tree
scribe q tree --latest --source claude-code --role subagent --table

# 2. Count subagents by fine_role
scribe q tree --latest --source claude-code --role subagent \
  --jq '[.turns[].requests[] | recurse(.children[]?) | select(.execution_role=="subagent") | .fine_role] | group_by(.) | map({role: .[0], count: length})'

# 3. Inspect a specific subagent's tools and config
scribe q show <subagent_id> --fields tools,config
# Shows: tool_count, tool_definitions, and model/max_tokens/temperature

# 4. Find what the longest subagent chain did
scribe q tree --latest --source claude-code \
  --jq '.turns[].requests[] | select(.children | length > 3) | .trace_request_id'
scribe q show <chain_root_id> --fields messages --full
```

## Track CLAUDE.md / memory injection

**Goal**: See when and how memory files get loaded into the system prompt.

```bash
# 1. Search for memory references
scribe q search "memory/" --role system --latest --source claude-code --full

# 2. Track memory changes across turns
scribe q tree --latest --source claude-code \
  --jq '[.turns[].requests[] | select(.execution_role=="primary") | .trace_request_id]'

# For each primary request ID:
scribe q diff <turn1_id> --fields memory
scribe q diff <turn2_id> --fields memory
scribe q diff <turn3_id> --fields memory
```

## Debug a tool call failure

**Goal**: A tool call returned an error or unexpected result. Find out what
happened.

```bash
# 1. Find the response that contains the error
scribe q search "error" --direction response --latest --role assistant

# 2. Show the full response
scribe q show <response_id> --full

# 3. Find the corresponding request (what was sent)
scribe q show <response_id>
# Note paired_trace_request_id
scribe q show <request_id> --fields messages --full \
  | jq '.messages[-3:]'  # Last few messages leading to the error
```

## Analyze token usage growth

**Goal**: Track how input tokens grow across a conversation.

```bash
# 1. Get primary request IDs in order
scribe q tree --latest --source claude-code \
  --jq '[.turns[].requests[] | select(.execution_role=="primary") | .trace_request_id]'

# 2. Show usage for each (skip the baseline)
scribe q diff <turn2_id> --fields usage
scribe q diff <turn3_id> --fields usage
# Each shows current vs previous input_tokens and the delta
```

## Find what triggered a model switch

**Goal**: The session started with one model and switched to another. When and why?

```bash
# 1. See model progression in the tree
scribe q tree --latest --source claude-code \
  --jq '.turns[].requests[] | select(.execution_role=="primary") | {turn: .turn_number, model: .model, id: .trace_request_id}'

# 2. When you spot the change, diff the config
scribe q diff <id_after_switch> --fields config
# Shows: {"model": {"from": "claude-sonnet-4-6", "to": "claude-opus-4-6"}}

# 3. Or inspect any request's config directly
scribe q show <id> --fields config
# Shows: {"config": {"model": "claude-opus-4-6", "max_tokens": 64000, "thinking": {"type": "adaptive"}}}
```

## Extract all tool definitions from a session

**Goal**: Get the complete list of tools an agent had access to.

```bash
# Find the first primary request (has all initial tools)
scribe q tree --latest --source claude-code \
  --jq '.turns[0].requests[0].trace_request_id'

# Extract tool names and descriptions
scribe q show <id> --fields tools --full \
  | jq '.tool_definitions[] | {name, description: .description[:80]}'
```

## Compare Codex sessions

**Goal**: Codex creates parallel sessions for each file. Compare them.

```bash
# 1. List recent Codex sessions
scribe q sessions --source codex --limit 10 --table

# 2. Show tree for each
scribe q tree --session <session1> --table
scribe q tree --session <session2> --table

# 3. Search for specific patterns across all Codex sessions
scribe q search "apply_patch" --source codex --role assistant
```

## Deep-dive a subagent type

**Goal**: Understand the full context a specific subagent type receives — its system
prompt, available tools, injected context, and how it differs from the primary.

```bash
# 1. Find subagent requests by type
scribe q tree --latest --source claude-code --fine-role task_subagent_explore --table
# Or search by system prompt fingerprint:
scribe q search "file search specialist" --role system --source claude-code --latest --limit 1

# 2. Extract the subagent's full system prompt
scribe q show <subagent_id> --fields messages --full \
  --jq '.messages[] | select(.role=="system") | .content'

# 3. Extract its tool list (use DB fallback if show --fields tools is empty)
sqlite3 ~/.scribe/traces.db \
  "SELECT body FROM raw_payloads WHERE trace_request_id = '<subagent_full_id>'" \
  | python3 -c "import json,sys; [print(t['name']) for t in json.loads(sys.stdin.read()).get('tools',[])]"

# 4. Compare tools: primary vs subagent
diff <(sqlite3 ~/.scribe/traces.db \
  "SELECT body FROM raw_payloads WHERE trace_request_id = '<primary_id>'" \
  | python3 -c "import json,sys; [print(t['name']) for t in json.loads(sys.stdin.read()).get('tools',[])]" | sort) \
  <(sqlite3 ~/.scribe/traces.db \
  "SELECT body FROM raw_payloads WHERE trace_request_id = '<subagent_id>'" \
  | python3 -c "import json,sys; [print(t['name']) for t in json.loads(sys.stdin.read()).get('tools',[])]" | sort)

# 5. Check what system-reminder context the subagent receives
scribe q show <subagent_id> --fields messages --full \
  --jq '.messages[] | select(.role=="user") | .content' \
  | grep -oP '<system-reminder>.*?</system-reminder>'

# 6. Check the Agent tool call that spawned it (from primary's response)
scribe q show <primary_id> --fields messages --full \
  --jq '.messages[] | select(.content | test("Agent")) | {content, tool_call_json}'
```

## Extract Agent tool call parameters from a response

**Goal**: See exactly what `subagent_type`, `model`, `prompt` the primary passed
when spawning subagents.

```bash
# 1. Find the primary request that spawned subagents
scribe q tree --latest --source claude-code --table
# Look for a primary with -> tool_use that has subagent children

# 2. Find the paired response
sqlite3 ~/.scribe/traces.db "
  SELECT id, direction FROM trace_requests
  WHERE request_id = (SELECT request_id FROM trace_requests WHERE id LIKE '<primary_id>%')
  ORDER BY direction"
# The 'response' row is what you need

# 3. Parse Agent tool_use blocks from the SSE stream
sqlite3 ~/.scribe/traces.db "SELECT body FROM raw_payloads WHERE trace_request_id = '<response_id>'" \
  | python3 -c "
import json, sys
text = sys.stdin.read().strip()
current_tool = None; current_json = ''
for line in text.split('\n'):
    line = line.strip()
    if not line.startswith('data: '): continue
    try:
        d = json.loads(line[6:])
        if d.get('type') == 'content_block_start':
            cb = d.get('content_block', {})
            if cb.get('type') == 'tool_use':
                if current_tool and current_tool.get('name') == 'Agent':
                    try: current_tool['input'] = json.loads(current_json)
                    except: pass
                    print(json.dumps(current_tool, indent=2, ensure_ascii=False))
                    print('---')
                current_tool = {'name': cb.get('name'), 'id': cb.get('id')}
                current_json = ''
        elif d.get('type') == 'content_block_delta':
            delta = d.get('delta', {})
            if delta.get('type') == 'input_json_delta':
                current_json += delta.get('partial_json', '')
    except: pass
if current_tool and current_tool.get('name') == 'Agent':
    try: current_tool['input'] = json.loads(current_json)
    except: pass
    print(json.dumps(current_tool, indent=2, ensure_ascii=False))
"
```

Example output:
```json
{
  "name": "Agent",
  "id": "toolu_01XYZ...",
  "input": {
    "description": "Research rs/zerolog",
    "model": "sonnet",
    "prompt": "Research the Go logging library rs/zerolog. Find:\n1. Latest version..."
  }
}
```

## Cross-reference scribe traces with CC local files

**Goal**: Correlate API-level trace data with Claude Code's local session
persistence (subagent metadata, agentId, conversation history).

```bash
# 1. Find the CC session UUID (appears in system-reminder as sessionId or in the JSONL)
#    CC uses UUIDs (a2839f01-...), scribe uses ULIDs (019d283ff76b...)
ls ~/.claude/projects/-Users-*-<project_name>/

# 2. List all subagents in a CC session
for f in ~/.claude/projects/.../<session_uuid>/subagents/*.meta.json; do
    echo "$(basename $f): $(cat $f)"
done

# 3. Map agentId to scribe trace data
#    agentId format: 17-char hex (e.g., a326d912dc8360845)
#    Search scribe for the subagent's system prompt fingerprint
scribe q search "file search specialist" --source claude-code --latest --limit 1
# The matching trace_request_id is the subagent's first API request

# 4. Compare: subagent JSONL (CC local) vs scribe trace (API level)
#    CC local has: hook progress events, isSidechain flag, agentId on every record
#    Scribe has: raw API payloads, tool definitions, usage stats, request tree
cat ~/.claude/projects/.../<session>/subagents/agent-<agentId>.jsonl | python3 -c "
import json, sys
for line in sys.stdin:
    d = json.loads(line.strip())
    print(f'{d.get(\"type\"):12} isSidechain={d.get(\"isSidechain\")} agentId={d.get(\"agentId\",\"?\")[:8]}')
"

# 5. Extract usage stats from parent JSONL (not available in scribe)
grep -o 'agentId: a[a-f0-9]*.*</usage>' ~/.claude/projects/.../<session>.jsonl
```

## Analyze system-reminder injection patterns

**Goal**: Understand what context CC injects into user messages and tool_result
messages across conversation rounds.

```bash
# 1. Check how many system-reminders a request has
scribe q show <id> --fields messages --full | python3 -c "
import json, sys, re
data = json.load(sys.stdin)
for m in data.get('messages', []):
    role = m.get('role','')
    c = json.dumps(m.get('content',''))
    reminders = re.findall(r'<system-reminder>(.*?)</system-reminder>', c, re.DOTALL)
    if reminders:
        print(f'[{role}] {len(reminders)} reminder(s)')
        for i, r in enumerate(reminders):
            print(f'  [{i}]: {r.strip()[:100]}...')
"

# 2. Track reminder injection across rounds (subagent multi-turn)
#    Find all rounds of a subagent conversation
sqlite3 ~/.scribe/traces.db "
WITH RECURSIVE chain(id, depth) AS (
    SELECT '<first_subagent_request_id>', 0
    UNION ALL
    SELECT tr.id, c.depth + 1
    FROM trace_requests tr JOIN chain c ON tr.parent_request_id = c.id
    WHERE tr.direction = 'request'
)
SELECT id, depth FROM chain ORDER BY depth"

# 3. For each round, check what reminders appear in tool_result messages
#    Key patterns:
#    - Explore: READ-ONLY reminder in almost every tool_result
#    - General: MCP instructions in first tool_result only
#    - Primary: claudeMd + hooks + skills + MCP in initial user message
```

## Best practices

1. **Always start with `tree`** to orient yourself. It's the map of the session.

2. **Use `search` to find, `show` to read, `diff` to compare.** Don't try to
   use one command for everything.

3. **Chain with jq** for programmatic analysis. Every command outputs JSON that
   jq can process.

4. **Use `--full` sparingly.** Default truncation keeps output manageable.
   Only use `--full` when you actually need to see the complete content.

5. **Save IDs from tree output.** The trace_request_id is the universal key
   that links all commands together.
