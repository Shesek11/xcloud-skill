---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Trigger a git deployment for an xCloud site via the API (without a new push) and view or update deploy settings — branch, push-deploy, deploy script. Use to redeploy the current branch, re-run the deploy script, or change deployment config.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## xCloud Deploy

See `plugins/xcloud/references/api-reference.md`. This complements `/xcloud-push`: use **`/xcloud-push`** to commit+push (webhook triggers deploy); use **`/xcloud-deploy`** to redeploy the *current* server branch without any new commit.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:
- **Trigger deploy** (default) — run the deployment now, then watch events
- **View settings** — show current git/deploy config
- **Update settings** — change branch, push-deploy, deploy script, etc.

### Step 3a: View settings

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/git"
```

Render: repository, `git_branch`, push-deploy enabled?, `run_after_deployment`, deploy script summary. **Never print `env_file_content`** — it contains secrets (DB password etc.). Redact it.

### Step 3b: Trigger deploy

Confirm first (which branch will deploy). Then:

```bash
curl -sS -X POST -w "\n__HTTP__:%{http_code}" -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/git/deploy"
```

Expect **202 (queued)**. Then poll events until the deploy finishes:

```bash
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/events?per_page=10"
```

Poll every ~5s for up to a few minutes. On completion, fetch the deploy log to confirm success:

```bash
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/deployment-logs"
```

If the deploy log shows errors, summarize them and suggest fixes (same logic as `/xcloud-deploy-log`).

### Step 3c: Update settings

Confirm the change, then PUT (send the full intended set of fields — this replaces deploy config):

```bash
curl -sS -X PUT -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"git_branch":"main","enable_push_deploy":true,"run_after_deployment":true,"deploy_script":"npm ci && npm run build && pm2 reload all","restart_services":true,"env_file_path":".env"}' \
  "$B/sites/$SITE_UUID/git"
```

After updating, optionally trigger a deploy to apply.

## Important Notes

- `POST /git/deploy` is **async (202)**; the response means "queued", not "done". Always verify via events + deployment-logs.
- This deploys whatever is on the configured branch on the server side — it does NOT push your local commits. To ship local changes, use `/xcloud-push`.
- `PUT /git` changes how every future deploy behaves — confirm and echo the new settings back to the user.
- `git.env_file_content` is sensitive; redact it in all output and never store it in `context.md`.
