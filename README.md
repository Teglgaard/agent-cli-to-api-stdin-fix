# stdin Fix for agent-cli-to-api

**Fixes: `[Errno 7] Argument list too long` when passing large prompts**

**Bonus: Adds cursor-agent token usage reporting**

This repository contains a minimal fix for the ARG_MAX limitation in [agent-cli-to-api](https://github.com/dev-thug/agent-cli-to-api).

## Problem

1. **ARG_MAX limit:** Linux systems have an ARG_MAX limit (~128 KB) for command-line arguments. When passing large prompts as argv, the subprocess call fails with `[Errno 7] Argument list too long`.

2. **Missing token usage:** cursor-agent token usage (prompt_tokens, completion_tokens, cache tokens) was not being extracted and returned in API responses.

3. **Streaming usage not reported:** Token usage was not included in the final streaming chunk, breaking OpenClaw and other OpenAI-compatible clients.

## Solution

1. Pass prompts via stdin instead of argv - bypassing ARG_MAX completely.

2. Add `extract_usage_from_cursor_agent_result()` to properly extract and report token usage from cursor-agent, including cache hit/miss tracking.

3. Include usage in final streaming chunk for OpenClaw/OpenAI compatibility.

## Files

**For PR to upstream:**
- `codex_gateway/stream_json_cli_stdin.py` - Core fix (replaces stream_json_cli.py) + token usage extraction
- `test_integration_stdin.py` - Integration tests for stdin fix
- `test_cursor_agent_usage.py` - Unit tests for token usage (optional)
- `test_cursor_agent_api_integration.py` - API integration tests for token usage (optional)
- `PR.md` - Pull request description

**Changes needed in upstream server.py:**
```python
# 1. Add token usage import
+ from .stream_json_cli_stdin import extract_usage_from_cursor_agent_result

# 2. Change import
- from .stream_json_cli import iter_stream_json_events
+ from .stream_json_cli_stdin import iter_stream_json_events

# 3. Remove cmd.append(prompt) and add stdin_data parameter (6 locations)
- cmd.append(prompt)
- async for evt in iter_stream_json_events(cmd=cmd, ...):
+ async for evt in iter_stream_json_events(cmd=cmd, stdin_data=prompt, ...):

# 4. Add usage extraction for cursor-agent (non-streaming)
  extract_cursor_agent_delta(evt, assembler)
+ maybe_usage = extract_usage_from_cursor_agent_result(evt)
+ if maybe_usage:
+     usage = maybe_usage

# 5. Add usage extraction for cursor-agent (streaming)
  delta = extract_cursor_agent_delta(evt, assembler)
+ maybe_usage = extract_usage_from_cursor_agent_result(evt)
+ if maybe_usage:
+     stream_usage = maybe_usage

# 6. Add usage to final streaming chunk (OpenClaw compatibility)
  end = {...}
+ if stream_usage:
+     end["usage"] = stream_usage
```

Total: **2 import changes + 6 call site changes + 8 usage extraction/reporting lines = ~20 lines**

## Testing

```bash
python3 test_integration_stdin.py
```

All tests should pass.

## Status

✅ Tested in production  
✅ Ready for upstream PR  
✅ Minimal changes (one new file + modifications to server.py)
