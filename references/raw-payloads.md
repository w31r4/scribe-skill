# Working with Raw Payloads

`scribe q raw` is the escape hatch — it returns the exact bytes stored in the
database, without extraction or interpretation. Use it when `show` doesn't expose
what you need.

## When to use raw vs show

| Need | Use |
|------|-----|
| Read messages, tools, usage | `show` (decoded, structured) |
| See SSE streaming events | `raw` (event: / data: lines) |
| Inspect content block structure | `raw --pretty` |
| Debug a parsing issue | `raw` (see what extractor sees) |
| Cursor protobuf bytes | `raw --decode-cursor` |
| Provider-specific fields | `raw --pretty` (e.g., `cache_creation_input_tokens`) |

## Provider-specific formats

### Anthropic (Claude Code)

**Requests** are JSON:
```bash
scribe q raw <request_id> --pretty
```
```json
{
  "model": "claude-opus-4-6",
  "max_tokens": 16384,
  "system": [{"type": "text", "text": "You are Claude Code..."}],
  "messages": [...],
  "tools": [...],
  "temperature": 1.0
}
```

**Responses** are SSE streams:
```bash
scribe q raw <response_id>
```
```
event: message_start
data: {"type":"message_start","message":{"model":"claude-opus-4-6",...}}

event: content_block_start
data: {"type":"content_block_start","index":0,...}

event: content_block_delta
data: {"type":"content_block_delta","delta":{"type":"text_delta","text":"Hello"}}

event: message_delta
data: {"type":"message_delta","delta":{"stop_reason":"end_turn"},...}
```

SSE responses are plain text, not JSON. `--pretty` has no effect. Use `--jq`
only on JSON payloads.

### OpenAI (Codex)

**Requests** are JSON with OpenAI format:
```bash
scribe q raw <request_id> --pretty
```
```json
{
  "model": "gpt-5",
  "messages": [{"role": "developer", "content": "..."}],
  "tools": [...],
  "temperature": 0.7
}
```

**Responses** are SSE streams similar to Anthropic but with OpenAI event format.

### Cursor

Cursor traffic is ConnectRPC over HTTP/2, encoded as protobuf.

```bash
# Default: try JSON parse, fall back to base64 wrapper
scribe q raw <cursor_id>

# Decode protobuf to JSON
scribe q raw <cursor_id> --decode-cursor
```

Without `--decode-cursor`, non-JSON payloads appear as:
```json
{"_base64": "CgRoZWxsby..."}
```

With `--decode-cursor`, the protobuf is decoded into a JSON structure.
The decoding quality depends on the proto schema coverage in
`scribe/cursor/proto_decoder.py`.

## Finding the paired request/response

Every request has a matching response (and vice versa). Use `show` to find the
pairing:

```bash
scribe q show <request_id>
# Look for "paired_trace_request_id" in the output

scribe q raw <paired_trace_request_id> --pretty
```

## Extracting specific data from raw JSON

For JSON payloads, combine `raw` with `--jq`:

```bash
# Get the model from a request
scribe q raw <id> --jq '.model'

# Count tools in a request
scribe q raw <id> --jq '.tools | length'

# Get system prompt text
scribe q raw <id> --jq '.system[].text'

# Get stop reason from an SSE response — won't work (SSE is not JSON)
# Use `show` instead for responses
scribe q show <response_id> --fields messages --jq '.messages[-1]'
```

## Debugging tips

### Payload is empty or null
The raw payload might be missing if:
- The request was a probe (`max_tokens=1`) — some probes don't store payloads
- The Cursor collector hit a late-body issue (payload arrived after flush)
- The proxy had a transient error during body capture

### Can't parse the response
SSE responses are text, not JSON. If you need structured data from a response:
```bash
# Use show instead — it reassembles SSE into structured messages
scribe q show <response_id> --fields messages,usage
```

### Unexpected content type
Check what the trace_request metadata says:
```bash
scribe q show <id>
# Look at "provider" and "direction" fields
```
