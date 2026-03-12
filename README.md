# stdin Fix for agent-cli-to-api

**Fixes: `[Errno 7] Argument list too long` when passing large prompts**

This repository contains a minimal fix for the ARG_MAX limitation in [agent-cli-to-api](https://github.com/dev-thug/agent-cli-to-api).

## Problem

Linux systems have an ARG_MAX limit (~128 KB) for command-line arguments. When passing large prompts as argv, the subprocess call fails with `[Errno 7] Argument list too long`.

## Solution

Pass prompts via stdin instead of argv - bypassing ARG_MAX completely.

## Files

**For PR to upstream:**
- `codex_gateway/stream_json_cli_stdin.py` - Core fix (replaces stream_json_cli.py)
- `test_integration_stdin.py` - Integration tests
- `PR.md` - Pull request description

**Changes needed in upstream server.py:**
```python
# Change import
- from .stream_json_cli import iter_stream_json_events
+ from .stream_json_cli_stdin import iter_stream_json_events

# Remove cmd.append(prompt) and add stdin_data parameter
- cmd.append(prompt)
- async for evt in iter_stream_json_events(cmd=cmd, ...):
+ async for evt in iter_stream_json_events(cmd=cmd, stdin_data=prompt, ...):
```

Total: **1 import change + 6 call site changes**

## Testing

```bash
python3 test_integration_stdin.py
```

All tests should pass.

## Status

✅ Tested in production  
✅ Ready for upstream PR  
✅ Minimal changes (one new file + modifications to server.py)
