# stdin Fix for agent-cli-to-api

**Fixes: `[Errno 7] Argument list too long` when passing large prompts**

**Bonus: Adds cursor-agent token usage reporting**

This repository contains a minimal fix for the ARG_MAX limitation in [agent-cli-to-api](https://github.com/dev-thug/agent-cli-to-api).

## Problem

1. **ARG_MAX limit:** Linux systems have an ARG_MAX limit (~128 KB) for command-line arguments. When passing large prompts as argv, the subprocess call fails with `[Errno 7] Argument list too long`.

2. **Missing token usage:** cursor-agent token usage (prompt_tokens, completion_tokens, cache tokens) was not being extracted and returned in API responses.

3. **Streaming usage not reported:** Token usage was not included in the final streaming chunk, breaking OpenClaw and other OpenAI-compatible clients.

4. **Rate limit failover broken:** Rate limit errors returned HTTP 500 with `codex_gateway_error`, preventing clients from treating limits as HTTP 429 / `rate_limit_error`.

5. **Claude Code interactive approvals:** Headless runs could stall on shell permission prompts; the gateway now passes `--dangerously-skip-permissions` for Claude CLI invocations.

## Solution

1. Pass prompts via stdin instead of argv - bypassing ARG_MAX completely.

2. Add `extract_usage_from_cursor_agent_result()` to properly extract and report token usage from cursor-agent, including cache hit/miss tracking.

3. Include usage in final streaming chunk for OpenClaw/OpenAI compatibility.

4. Automatically detect rate limit errors and return HTTP 429 with `rate_limit_error` type.

5. Claude Code: add `--dangerously-skip-permissions` on the CLI command line (non-streaming + streaming paths).

**OpenClaw:** To actually move to the next entry in `model.fallbacks` on 429, use **separate `models.providers` entries** for primary vs fallback (same `baseUrl` is OK). See [Model failover](https://docs.openclaw.ai/concepts/model-failover) and `RATE_LIMIT_FAILOVER_FIX.md`.

## Files

**This repository (fully up to date with our fork):**
- `codex_gateway/stream_json_cli_stdin.py` - stdin streaming + cursor-agent usage extraction
- `codex_gateway/server.py` - **full patched file** (stdin, usage, streaming usage, 429 mapping, Claude flags)
- `codex_gateway/openai_compat.py` - dependency for the stdin module tests
- `codex_gateway/__init__.py` - package marker

**Tests:**
- `test_integration_stdin.py` - Integration tests for stdin fix
- `test_cursor_agent_usage.py` - Unit tests for token usage (optional)
- `test_cursor_agent_api_integration.py` - API integration tests for token usage (optional)
- `test_rate_limit_failover.py` - Tests for rate limit detection (optional)
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

# 7. Add rate limit detection in _openai_error() for failover
  def _openai_error(message: str, *, status_code: int = 500):
+     # Detect rate limit errors for OpenClaw failover
+     if any(phrase in message.lower() for phrase in [
+         "hit your limit", "rate limit", "too many requests",
+         "quota exceeded", "usage limit"
+     ]):
+         error_type = "rate_limit_error"
+         error_code = "rate_limit_exceeded"
+         status_code = 429

# 8. Claude CLI cmd: add "--dangerously-skip-permissions" (both Claude call sites)
```

See `codex_gateway/server.py` in this repo for the complete, tested patch. Summary: **stdin + cursor usage + final-chunk usage + 429 errors + Claude non-interactive CLI flags**.

## Testing

```bash
python3 test_integration_stdin.py
```

All tests should pass.

## Status

✅ Tested in production  
✅ Ready for upstream PR  
✅ Minimal changes (one new file + modifications to server.py)
