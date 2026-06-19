---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: View Google PageSpeed Insights scores for an xCloud site (mobile + desktop), browse score history, or trigger a fresh scan. Use to track Core Web Vitals and performance over time.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## xCloud PageSpeed

See `references/api-reference.md` for the auth helper.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:
- **Latest** (default) — most recent mobile + desktop snapshot
- **History** — past scores (choose strategy: mobile or desktop)
- **Scan now** — trigger a new PageSpeed Insights scan (async)

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"
B="https://app.xcloud.host/api/v1"

curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/pagespeed"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/pagespeed/history?strategy=mobile&per_page=20"
curl -sS -X POST -w "\n__HTTP__:%{http_code}" -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/pagespeed/scan"
```

### Step 4: Render

- **Latest** returns `{mobile, desktop}`. If both are `null`, no scan has run yet — offer "Scan now".
- Show performance score plus Core Web Vitals (LCP, CLS, etc.) when present. Flag mobile score < 50 (poor) / 50–89 (needs improvement) / 90+ (good).
- **History** — compact table of date + score so trends are visible.

### Step 5: After a scan

`POST /pagespeed/scan` is **async (202)**. Tell the user results take a minute or two; re-run "Latest" to see them. Don't poll aggressively.

## Important Notes

- Scores come from Google PageSpeed Insights and reflect the public URL — make sure the site is reachable.
- `strategy` is `mobile` or `desktop`. Mobile is usually the stricter (and more important) score.
- For server/site resource usage (CPU/RAM), use `/xcloud-monitoring` instead.
