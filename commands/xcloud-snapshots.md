---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: List site snapshots on xCloud — for a single site or for every site on a server. Use to see what restore points exist before a risky change. (Snapshots are listed via the API; creating and restoring are done in the xCloud panel.)
---

## Context

- Site UUID:   !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`
- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`

## xCloud Snapshots

See `plugins/xcloud/references/api-reference.md` for the auth helper.

### Step 1: Resolve UUIDs

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick scope

Use AskUserQuestion:
- **This site** (default) — snapshots for the current site
- **Whole server** — snapshots across all sites on the server

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"
B="https://app.xcloud.host/api/v1"

curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/snapshots?per_page=50"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/snapshots?per_page=50"
```

### Step 4: Render

Table: `# | name/label | created | size | site | uuid`. If the list is empty, say so and point out that a WordPress site can also be **created from a snapshot** via `snapshot_uuid` in `POST /servers/{uuid}/sites/wordpress`.

## Important Notes

- The public API exposes **listing** snapshots only — there is no create/restore/delete snapshot endpoint. Do those in the xCloud panel.
- A snapshot UUID from here can seed a new WordPress site (see `/servers/{uuid}/sites/wordpress` → `snapshot_uuid`).
- For routine point-in-time recovery, prefer `/xcloud-backup` (which can be triggered via the API).
