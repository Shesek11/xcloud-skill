---
allowed-tools: Bash(git:*), Bash(ssh:*), Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(sleep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Write, Glob, Grep, AskUserQuestion
description: Commit changes and push to GitHub. If auto-deploy is configured, watches xCloud deployment events post-push to verify the deploy actually succeeded — not just that git push returned 0.
---

## Context

- Current directory: !`pwd`
- Git status: !`git status --short 2>/dev/null`
- Current branch: !`git branch --show-current 2>/dev/null`
- Unpushed commits: !`git log origin/$(git branch --show-current 2>/dev/null)..HEAD --oneline 2>/dev/null | head -5`
- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Push Changes & Verify Deployment

### Step 1: Verify state

1. Confirm in a git repo.
2. Read `.xcloud/context.md` for branch, deploy-branch, push-deploy flag, Site UUID.
3. If no Site UUID in context (legacy format), suggest `/xcloud-init` migration. Push will still work, but post-push verification will be SSH-based instead of API-based.

### Step 2: Review changes

Show:
- Unstaged
- Staged
- Untracked
- Unpushed commits

### Step 3: Stage and commit (if needed)

If uncommitted changes exist, AskUserQuestion:
- **All Changes** — stage everything
- **Select Files** — user specifies
- **Only Staged** — commit existing staged files only
- **Skip Commit** — push existing commits as-is

Ask for commit message or auto-generate from diff.

```bash
git add [files]
git commit -m "[message]"
```

### Step 4: Pre-push safety

Check for sensitive files:

```bash
git diff --cached --name-only | grep -E '\.(env|pem|key)$|credentials|secrets'
```

If pushing to deploy-branch with push-deploy enabled:

```
WARNING: Pushing to [branch] will trigger auto-deploy to LIVE [domain_name].
Continue? (must confirm)
```

### Step 5: Push

```bash
git push origin [branch]
```

Capture the new HEAD commit hash for verification:

```bash
PUSHED_HASH=$(git rev-parse HEAD)
PUSHED_SHORT=$(git rev-parse --short HEAD)
```

### Step 6: Verify deployment via API (if push-deploy enabled)

Watch deployment via the API — much more reliable than SSH polling because we get the actual deploy script output and exit status.

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"
H_ACC="Accept: application/json"
BASE="https://app.xcloud.host/api/v1"

# Wait a few seconds for the webhook → deploy chain to start
sleep 8

# Poll deployment-logs every 5s for up to 5 minutes, looking for our pushed commit
START_TS=$(date +%s)
DEADLINE=$((START_TS + 300))

while [ $(date +%s) -lt $DEADLINE ]; do
  LOGS=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/deployment-logs?per_page=5")
  # Look for an entry where commit_hash starts with $PUSHED_SHORT
  # Status field tells us: pending / running / completed / failed
  # If we find one with status completed → success; failed → fail; otherwise keep polling
  sleep 5
done
```

Show progress:

```
[00:08] Polling deployment-logs for [PUSHED_SHORT]...
[00:13] Found deploy: status=running
[00:18] Still running...
[00:25] status=completed ✓
[00:25] Deploy verified — [PUSHED_SHORT] is live.
```

### Step 7: On verification failure

If after 5 minutes no matching deploy:

```
WARNING: No deployment for [PUSHED_SHORT] appeared in xCloud within 5 minutes.
Possible causes:
  • Push-deploy webhook not firing (check git push response, GitHub webhook delivery)
  • xCloud panel: Sites → [domain] → Git → push-deploy is OFF
  • Webhook IP blocked

Manual recovery:
  curl -sS -H "..." "$BASE/sites/$SITE_UUID/git"   # check enable_push_deploy
  /xcloud-ssh   # SSH in and run the deploy script manually
```

If a deploy with status `failed` appeared:

```
DEPLOY FAILED for [PUSHED_SHORT]:
[fetch the log body and show last 30 lines]

See `/xcloud-deploy-log` for full output and suggested fixes.
```

### Step 8: Update context

Append to `## Development Log`:

```markdown
### [DATE] [TIME] - Git Push
- Branch: [branch]
- Commit: [hash] [msg]
- Files changed: [count]
- Push-deploy: [enabled/disabled]
- Verification: [verified ok in Xs / FAILED / not verified]
```

If verified, also append to `## Recent Deployments`:

```markdown
| [date] | [hash] | [summary] | Verified OK |
```

## Important Notes

- **Do not trust `git push` exit 0 as deploy success.** xCloud may silently fail (push-deploy off, deploy script error). Always verify via the API.
- API verification is faster and more reliable than SSH-ing in: it catches build errors, runtime errors, and webhook misconfiguration.
- For sites with push-deploy disabled, this command falls through to a SSH-based check (legacy behavior).
- Never force push without explicit user confirmation. Skip the verify step on force-push (deploy may not be triggered for force-push depending on xCloud config).
