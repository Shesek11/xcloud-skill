---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Write, Glob, AskUserQuestion
description: List backups for an xCloud site, trigger a new backup, or inspect backup settings/status/count. Use before risky changes (DB migrations, plugin updates) or to verify the backup schedule is healthy.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Site Backups

See `references/api-reference.md` for the auth helper.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. If absent, run `/xcloud-init` first.

### Step 2: Pick action

Use AskUserQuestion:
- **List backups** (default) — show history
- **Status & settings** — last-backup state + local/remote schedule + count
- **Trigger backup** — start a new one (async)
- **Trigger then watch** — start and poll events until done

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"
B="https://app.xcloud.host/api/v1"

# List history
curl -sS -w "\n__HTTP__:%{http_code}" -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/backups?per_page=20"
# Status / settings / count
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/backup-status"     # {local:{configured,active,status,last_backup_at}, remote:{...}}
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/backup-settings"   # schedule/retention config
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/backup-count"      # total number of backups
```

Render the history as a table (parse fields defensively — likely `created_at`, `type`, `size`, `status`):

```
| # | Date | Type | Size | Status |
|---|------|------|------|--------|
| 1 | 2026-06-17 03:00 | scheduled | 412 MB | ok |
| 2 | 2026-06-16 03:00 | scheduled | 408 MB | ok |
```

If a row has `status: failed`, surface the error. For **Status & settings**, show whether local and remote backups are configured/active, the last backup time, retention, and total count. Flag if `last_backup_at` is old or remote isn't configured.

### Step 4: Trigger backup

Confirm with the user before triggering:

```
This will start a backup for [domain_name].
Backups can take several minutes for large sites.
Proceed? (yes/no)
```

On confirm:

```bash
curl -sS -w "\n__HTTP__:%{http_code}" -X POST -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/backup"
```

Expected: HTTP **202** ("Backup queued"). If 403 → token lacks `write:sites`; tell the user to recreate it.

### Step 5: Trigger then watch

After a successful 202, poll `/sites/$SITE_UUID/events` every ~5s. Look for a `backup` completion event. Show progress and stop on completion, on `failed`, or after 10 minutes (whichever first):

```
[00:05] Queued.
[00:45] In progress...
[02:13] Completed. ✓
```

### Step 6: Update context

If a backup was triggered, append to `## Notes` (or a `## Backup History` section):

```markdown
### [DATE] [TIME] - Backup triggered
- Type: manual
- Status: completed (X MB)
```

## Important Notes

- Triggered backups are **async** (HTTP 202) — the response means "queued", not "done".
- The earlier beta-era 500 on `GET /backups` is fixed; if you ever do see a 5xx, surface it and check the panel rather than retrying in a loop.
- xCloud may rate-limit manual backups. If you hit 429, wait and retry.
- **Restore is NOT exposed via the API** — restore from the xCloud panel.
