---
allowed-tools: Bash(git:*), Bash(ssh:*), Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Write, Glob, AskUserQuestion
description: Comprehensive status of an xCloud project across local repo, GitHub remote, live server, AND xCloud API (deploys, SSL, last backup, monitoring). Use to spot drift or sync issues.
---

## Context

- Current directory: !`pwd`
- Git status: !`git status --short 2>/dev/null | head -20`
- Current branch: !`git branch --show-current 2>/dev/null`
- Last local commit: !`git log -1 --format="%h %s (%cr)" 2>/dev/null`
- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Show Project Status

Combines four sources: local git, GitHub remote, xCloud API, and (optionally) live SSH.

### Step 1: Read context

Locate `.xcloud/context.md`. If not found, run `/xcloud-init` first. Extract `Site UUID`, `Server UUID`, branch, deploy-branch, SSH host/user/port, remote path, db info.

If the file lacks UUIDs (legacy format), run `/xcloud-init` in migration mode — it will enrich the file without losing data.

### Step 2: Pick scope

Use AskUserQuestion:
- **Quick** — local git + GitHub remote only (no API, no SSH)
- **Standard** (default) — adds xCloud API: deploy status, SSL, last backup
- **Deep** — adds SSH: PM2, disk, memory, ecosystem config

### Step 3: Local + remote (always)

```bash
git fetch origin --quiet
git status --short
git log -1 --format='%h %s (%cr)'
git log HEAD..origin/[branch] --oneline   # commits to pull
git log origin/[branch]..HEAD --oneline   # commits to push
```

### Step 4: xCloud API (Standard + Deep)

Single round of parallel API calls:

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"
H_ACC="Accept: application/json"
BASE="https://app.xcloud.host/api/v1"

DEPLOY_LOGS=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/deployment-logs?per_page=1")
SSL=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/ssl")
BACKUPS=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/backups?per_page=1")
GIT=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/git")

# Live HTTP check (independent of xCloud — true end-user view)
LIVE_HTTP=$(curl -sI -o /dev/null -w "%{http_code}" --max-time 10 "https://[domain_name]")
```

Extract:
- Last deploy commit hash, status, duration (from `deployment-logs.items[0]`)
- SSL `provider`, `status`, `expires_at` → compute days remaining
- Last backup date + status (note: `/backups` may return 500 — handle gracefully, show "unavailable")
- Push-deploy enabled flag (`git.enable_push_deploy`)
- Live HTTP code (independent curl to public domain)

### Step 5: SSH (Deep only)

Re-use the existing single-SSH diagnostic pattern (PM2 list, reboot persistence, ecosystem config, disk, memory). Only run if user picked Deep AND SSH key auth works:

```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes -p 22 [user]@[ip] "echo SSH_KEY_OK" 2>/dev/null
```

If key auth not configured, skip SSH gracefully (do not prompt for password mid-status).

### Step 6: Display

```
============================================================
  PROJECT STATUS: [domain_name]   (type: [type])
============================================================

LOCAL REPOSITORY
----------------
Branch: [branch]                Status: [clean / X uncommitted]
Last:   [hash] [msg] ([rel-time])

GITHUB REMOTE
-------------
Ahead:  [N] commits to push
Behind: [N] commits to pull
Last:   [hash] [msg]

XCLOUD (API + live curl)
------------------------
Live HTTP:    [code]            (independent curl to https://[domain])
Last deploy:  [hash] @ [time]   ([finished/failed/running], [duration]) — or "no logs"
Last backup:  [date] [status]   — or "API 500 (beta issue), check panel"
SSL:          [provider], status [status], expires in [days] days  [⚠ if <14]
Push-deploy:  [enabled/disabled]

[only if Deep]
SSH PM2
-------
Processes:    [count] online / [errored]
Persistence:  [@reboot cron entries]
Ecosystem:    [primary config OK / dual config WARNING]
Disk:         [used/total]
Memory:       [used/total]

============================================================
```

### Step 7: Diagnose & suggest

In severity order:

**CRITICAL:**
- Live HTTP not 200 — site is down. Check `/xcloud-deploy-log` and `/xcloud-events`.
- Last deploy `status: failed` — fetch the log body and surface the error.
- SSL `expires_at` < 7 days OR `status: failed` — needs intervention; check `/xcloud-events` for renewal attempts.
- Server commit ≠ pushed commit — auto-deploy broken; suggest manual deploy via `/xcloud-ssh`.
- (Deep only) PM2 has 0 processes — Node app down.

**WARNING:**
- Pushable / pullable commits.
- Uncommitted local changes.
- SSL expiring 7–14 days.
- Last backup > 30 days old (or never).
- (Deep only) No `@reboot` persistence for Node.js.

**INFO:**
- Everything in sync.

### Step 8: Update context

Append to `## Development Log`:

```markdown
### [DATE] [TIME] - Status check ([scope])
- Local: [clean/X uncommitted]
- GitHub: [ahead X / behind X / sync]
- Last deploy: [hash] [status]
- SSL: [days] days
- HTTP: [code]
[deep only:]
- PM2: [count online]
- Disk: [used/total]
```

## Important Notes

- Standard mode does NOT touch SSH — it's API-only and fast (~2s for all calls in parallel).
- The API is the source of truth for "what's deployed" — more reliable than SSH-ing in to check git on the server (which can drift if a deploy partially failed).
- If the API and SSH disagree about the deployed commit, trust SSH (`git log -1`) — that's what's actually running.
