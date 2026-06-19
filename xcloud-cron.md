---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Manage cron jobs for an xCloud site or server — list, create, update, delete, run-now, and view last output. Use to schedule recurring tasks (wp-cron, cleanups, keepalives) or inspect existing ones.
---

## Context

- Site UUID:   !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`
- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`

## xCloud Cron Jobs

See `references/api-reference.md` for the auth helper.

### Step 1: Resolve UUIDs

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick scope + action

Use AskUserQuestion:
- **Scope**: Site cron vs Server cron (server cron also needs a `user`).
- **Action**: List (default) · Create · Update · Delete · Run now · View output

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"

# --- SITE cron ---
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/cron-jobs?per_page=50"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"frequency":"daily","pattern":"0 3 * * *","command":"wp cron event run --due-now"}' \
  "$B/sites/$SITE_UUID/cron-jobs"
curl -sS -X PUT    -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"frequency":"hourly","pattern":"0 * * * *","command":"..."}' "$B/sites/$SITE_UUID/cron-jobs/$CRON_UUID"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/cron-jobs/$CRON_UUID"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/cron-jobs/$CRON_UUID/execute"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/cron-jobs/$CRON_UUID/output"

# --- SERVER cron (note the extra "user" field) ---
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/cron-jobs"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"user":"site_user","frequency":"daily","pattern":"0 4 * * *","command":"/usr/bin/php /path/script.php"}' \
  "$B/servers/$SERVER_UUID/cron-jobs"
curl -sS -X PUT    -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"user":"site_user","frequency":"daily","pattern":"...","command":"..."}' "$B/servers/$SERVER_UUID/cron-jobs/$CRON_UUID"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/cron-jobs/$CRON_UUID"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/cron-jobs/$CRON_UUID/execute"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/cron-jobs/$CRON_UUID/output"
```

`frequency` is a friendly label (e.g. `daily`, `hourly`, `custom`); `pattern` is the actual cron expression. When creating, prefer setting both — for non-standard schedules use `frequency: custom` with an explicit `pattern`.

### Step 4: Render

List as a table: `# | command | frequency | pattern | last run/status | uuid`. For **View output**, show the captured stdout/stderr of the last run — useful to confirm a job actually works.

### Step 5: Confirm writes

Confirm before **Create / Update / Delete**. Echo the resolved `pattern` and `command` so the user can sanity-check the schedule. Deleting a cron is irreversible via the API.

## Important Notes

- Site cron runs as the site's system user; server cron requires you to name the `user`.
- "Run now" (`/execute`) is great for testing a job without waiting for the schedule.
- For Node.js keepalives, a server cron calling a shell script (e.g. `ensure-app-up.sh`) is the common pattern; see also the `xcloud-nodejs-persistence` skill.
