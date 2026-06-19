---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: View recent deployment logs for an xCloud site, including the actual deploy script output. Use to debug failed deploys or verify what was run.
---

## Context

- Site UUID from context: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Deployment Logs

### Step 1: Resolve site UUID

Read `.xcloud/context.md` → `Site UUID:`. Abort with `/xcloud-init` suggestion if missing.

### Step 2: Fetch logs

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/sites/$SITE_UUID/deployment-logs?per_page=20"
```

If the response is `data.items: []` (empty), tell the user:

```
No deployment logs found for [domain_name].
Possible reasons:
  • The site has never been deployed via xCloud's push-deploy.
  • Push-deploy is disabled (check /xcloud-info → Push-Deploy).
  • Logs were pruned (xCloud keeps a limited history).

If you've been deploying manually via SSH + git pull, those won't appear here.
```

### Step 3: Show summary list first

Don't dump all log bodies. Show a numbered list:

```
1. ✓ 2026-04-25 18:42  main @ a1b2c3d  (12s)  — completed
2. ✗ 2026-04-25 18:30  main @ 9f8e7d6  (3s)   — failed: "npm: command not found"
3. ✓ 2026-04-23 09:00  main @ 7c6b5a4  (15s)  — completed
...
```

### Step 4: Drill into one log

Use AskUserQuestion to offer:
- **#1 (latest)** — show full log body
- **Pick by number** — user types
- **All failures** — show full bodies of every failed deploy in the list
- **Cancel**

When showing a body:

```
─── Deployment #1 ───
Commit:    a1b2c3d "Fix navbar"
Branch:    main
Started:   2026-04-25 18:42:00
Finished:  2026-04-25 18:42:12
Duration:  12s
Status:    completed

─── Output ───
[full stdout/stderr from the deploy script]
```

### Step 5: Suggest fixes for common failures

Pattern-match the log body for known issues and propose the fix:

| Pattern in log | Likely cause | Suggested fix |
|---|---|---|
| `npm: command not found` | Node not in deploy script PATH | Add `export PATH=$PATH:/usr/local/bin` to deploy script |
| `Permission denied (publickey)` | SSH key mismatch on private repo | Re-add deploy key in xCloud panel |
| `npm ERR! peer dep` | dependency conflict | Remove `node_modules` and `package-lock.json`, re-deploy |
| `pm2: not found` | PM2 not installed for the user | `npm i -g pm2` in deploy script or login shell |
| `composer: command not found` | PHP-only path | Add composer install path or use `php /usr/local/bin/composer` |
| `out of memory` / `Killed` | Build OOM | Increase swap or build locally; see `/xcloud-monitoring` |

### Step 6: Update context

Append to `## Recent Deployments` in `.xcloud/context.md`:

```markdown
| Date | Commit | Status | Duration |
|------|--------|--------|----------|
| [date] | [hash] | [status] | [duration] |
```

Only add if not already present.

## Important Notes

- Read-only.
- Log bodies can be long (hundreds of lines for big builds). Default to summary view.
- If `enable_push_deploy: false` (check via `/xcloud-info`), there will be no deploy logs — the user is deploying manually via SSH.
