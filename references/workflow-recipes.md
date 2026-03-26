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
# 1. Get the tree
scribe q tree --latest --source claude-code

# 2. Count subagents by fine_role
scribe q tree --latest --source claude-code \
  --jq '[.turns[].requests[] | recurse(.children[]?) | select(.execution_role=="subagent") | .fine_role] | group_by(.) | map({role: .[0], count: length})'

# 3. Find what the longest subagent chain did
scribe q tree --latest --source claude-code \
  --jq '.turns[].requests[] | select(.children | length > 3) | .trace_request_id'
# Then inspect it:
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
