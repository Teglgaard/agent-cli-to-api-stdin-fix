# Push to GitHub and Create PR

## Step 1: Create GitHub Repository

1. Go to https://github.com/Teglgaard
2. Click "New repository"
3. Name: `agent-cli-to-api-stdin-fix`
4. Description: `Fix [Errno 7] Argument list too long by using stdin`
5. Public repository
6. **DO NOT** initialize with README, .gitignore, or license (we already have them)
7. Click "Create repository"

## Step 2: Push to GitHub

```bash
cd /Users/Teglgaard/Krown-Development/aws-krownwebservice/agent-cli-to-api-stdin-fix

# Add remote
git remote add origin https://github.com/Teglgaard/agent-cli-to-api-stdin-fix.git

# Push
git push -u origin master
```

## Step 3: Create PR to Upstream

1. Go to https://github.com/dev-thug/agent-cli-to-api
2. Click "Fork" (if you haven't already)
3. In your fork, click "Fetch upstream" to sync
4. Create a new branch:
   ```bash
   git checkout -b fix/stdin-argmax
   ```

5. Copy files from this repo:
   ```bash
   # Copy the core fix
   cp codex_gateway/stream_json_cli_stdin.py <path-to-fork>/codex_gateway/
   
   # Copy the test
   cp test_integration_stdin.py <path-to-fork>/
   ```

6. Edit `server.py` in the fork (13 line changes):
   - Change import: `from .stream_json_cli_stdin import`
   - Remove 6× `cmd.append(prompt)`
   - Add 6× `stdin_data=prompt` parameter

7. Commit changes:
   ```bash
   git add .
   git commit -F ../agent-cli-to-api-stdin-fix/PR.md
   git push origin fix/stdin-argmax
   ```

8. Go to https://github.com/dev-thug/agent-cli-to-api
9. Click "Pull requests" → "New pull request"
10. Click "compare across forks"
11. Select your fork and `fix/stdin-argmax` branch
12. Title: `Fix [Errno 7] Argument list too long by using stdin`
13. Description: Copy from `PR.md`
14. Click "Create pull request"

## Alternative: Direct PR from This Repo

If you want to reference this repo in the PR:

1. Push this repo to GitHub (steps 1-2 above)
2. In the PR description, add:
   ```
   Reference implementation: https://github.com/Teglgaard/agent-cli-to-api-stdin-fix
   ```

## Files in This Minimal Repo

```
.
├── README.md                              (1.4 KB) - Overview
├── PR.md                                  (2.2 KB) - PR description
├── codex_gateway/
│   ├── __init__.py                        (25 B)   - Module init
│   ├── openai_compat.py                   (1.4 KB) - Dependency
│   └── stream_json_cli_stdin.py           (8.0 KB) - Core fix
└── test_integration_stdin.py              (8.9 KB) - Tests

Total: 7 files, ~22 KB
```

## What Upstream Needs to Change

**In `codex_gateway/server.py`:**

1 import change + 12 line changes at 6 call sites = **13 total lines**

That's it! Minimal and focused.
