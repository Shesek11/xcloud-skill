---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Show recent activity timeline for an xCloud site (deployments, backups, config changes, etc.). Use to see what's happened or to confirm an async operation completed.
---

## Context

- Site UUID from context: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Site Event Timeline

### Step 1: Resolve site UUID

Read from `.xcloud/context.md`. If absent, run `/xcloud-init` first.

### Step 2: Fetch events

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/sites/$SITE_UUID/events?per_page=50"
```

### Step 3: Render

Each event has fields: `uuid`, `type`, `status`, `output` (potentially long text), `created_at`.

Treat `type` and `status` as opaque strings — observed values include `system`, `finished` (instead of `ok`/`completed`). Don't filter on assumed enum values; show what the API returns.

For each event, render a single line followed by a short snippet of `output`:

```
[created_at]  [type]    [status]   [first 80 chars of output, single line]
```

Example (real shapes):

```
2026-04-07 05:02:33  system   finished   Saving debug log to /var/log/letsencrypt/letsencrypt.log...
```

If `output` spans multiple lines, replace newlines with spaces and truncate. Offer to drill in (Step 4 → "Show full output for #N").

### Step 4: Filter & follow-up

Use AskUserQuestion to offer:
- **All events** (default — last 50)
- **Show full output for #N** — print the entire `output` field for one event
- **Filter by type substring** — user types e.g. `letsencrypt` or `deploy`; client-side filter
- **Failures only** — `status` is anything other than `finished` / `ok` / `completed`
- **Watch live** — re-fetch every 5s until user cancels (useful right after a push or trigger)

### Step 5: Watch mode (optional)

If user picks "Watch live":

```bash
LAST_TS=""
while true; do
  RESP=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/events?per_page=10")
  # Print only events with timestamp > LAST_TS
  # Update LAST_TS
  sleep 5
done
```

User can Ctrl+C to stop. Always tell them how to exit before starting.

## Important Notes

- Events are read-only.
- Pagination: `per_page=50` is enough for most users. Use `?page=2` to go further back.
- Treat unknown event `type` strings as opaque — the API may add new ones.
