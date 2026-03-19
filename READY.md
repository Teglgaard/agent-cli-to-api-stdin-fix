# ✅ Ready for GitHub & PR (fully synced)

## What We Have

**Repository** with the **full patched gateway** plus docs and tests (aligned with `agent-cli-to-api-local` / production server):

```
agent-cli-to-api-stdin-fix/
├── codex_gateway/
│   ├── __init__.py
│   ├── openai_compat.py          - Dependency for stdin module
│   ├── stream_json_cli_stdin.py  - stdin + cursor-agent usage extraction ⭐
│   └── server.py                 - Full patched server (stdin, usage, 429, Claude flags) ⭐
├── test_integration_stdin.py
├── test_cursor_agent_usage.py
├── test_cursor_agent_api_integration.py
├── test_rate_limit_failover.py
├── PR.md
├── README.md
├── RATE_LIMIT_FAILOVER_FIX.md
├── PUSH_TO_GITHUB.md
├── .gitignore
└── READY.md
```

## Fixes Included (server.py + stream_json_cli_stdin.py)

1. **stdin / ARG_MAX** – `stdin_data=prompt`, no `cmd.append(prompt)` for CLI backends  
2. **cursor-agent usage** – `extract_usage_from_cursor_agent_result` (non-stream + stream)  
3. **Streaming final chunk** – `usage` on last SSE chunk before `[DONE]`  
4. **Rate limits** – HTTP **429** + `type: rate_limit_error` + `code: rate_limit_exceeded` in `_openai_error`  
5. **Claude Code headless** – `--dangerously-skip-permissions` on Claude CLI invocations  

## OpenClaw

Gateway **429** is necessary but not sufficient: use **separate provider ids** for primary vs fallback when both models hit the same gateway. See `RATE_LIMIT_FAILOVER_FIX.md` § *OpenClaw configuration*.

## Upstream PR Options

- **Option A:** Open PR with only the minimal diff (new `stream_json_cli_stdin.py` + patch to upstream `server.py`) — copy snippets from `PR.md`.  
- **Option B:** Point reviewers at this repo’s `codex_gateway/server.py` for a full-file review.

## Your Next Actions

```bash
cd /Users/Teglgaard/Krown-Development/aws-krownwebservice/agent-cli-to-api-stdin-fix
git status
git add -A
git commit -m "Sync server.py, stream_json_cli_stdin, tests; docs for OpenClaw provider split"
git push origin master   # or your branch
```

*(Commit only when you’re ready — per your workflow.)*

## Tests

```bash
python3 test_integration_stdin.py
python3 test_cursor_agent_usage.py
python3 test_rate_limit_failover.py
# test_cursor_agent_api_integration.py may need venv + deps + mock setup
```

---

**Status: synced with local fork / server deployment** 🚀
