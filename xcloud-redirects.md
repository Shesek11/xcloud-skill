---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Inspect an xCloud site's URL and access configuration — redirections, web-rules (custom headers + redirects), and IP access rules. Use to audit how requests are routed, what headers are added, and who can reach the site.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## xCloud Redirects & Web Rules

See `references/api-reference.md` for the auth helper.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick what to inspect

Use AskUserQuestion (or show **All** by default):
- **Redirections** — `{items, count}`
- **Web rules** — custom headers + redirects: `{items, counts:{headers, redirects, total}}`
- **IP access** — allow/deny rules
- **Custom nginx** — raw custom nginx config blocks
- **Site scripts** — deploy/site scripts

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"
B="https://app.xcloud.host/api/v1"

curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/redirections"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/web-rules"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/ip-access"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/custom-nginx"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/site-scripts"
```

### Step 4: Render

- **Redirections** — table: `source → target | code (301/302) | uuid`.
- **Web rules** — split into Headers (name → value) and Redirects; use the `counts` block for totals.
- **IP access** — list allow/deny entries; flag if a deny-all is in place (could explain a site being unreachable).
- **Custom nginx / site scripts** — show the raw content in a fenced block; note these are advanced/server-managed.

## Important Notes

- These endpoints are **read-only** in the public API — to add or change a redirect/header/IP rule, use the xCloud panel (or edit nginx/site config over `/xcloud-ssh`).
- If a site is returning unexpected 301/302s or 403s, this command is the fastest way to see why.
- `domains` (primary + aliases) is shown by `/xcloud-info`; this command focuses on routing/header/access rules.
