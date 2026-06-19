---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: List staging environments for an xCloud site. Use to see which staging sites exist, their domains, and status before testing or pushing changes live. (Creating/syncing staging is done in the xCloud panel.)
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## xCloud Staging Sites

See `plugins/xcloud/references/api-reference.md` for the auth helper.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: List staging environments

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/sites/$SITE_UUID/staging-sites"
```

Response shape: `{ "items": [...], "count": N }`.

### Step 3: Render

Table: `# | staging domain | status | created | uuid`. If `count` is 0, tell the user there are no staging environments and that staging is created from the xCloud panel (Sites → [domain] → Staging).

Each staging environment is itself a site — its `uuid` works with the other read commands (`/xcloud-info`, `/xcloud-events`, `/xcloud-monitoring`) by setting it as the Site UUID.

## Important Notes

- The public API exposes **listing** staging sites only — create / sync-to-live / delete are panel operations.
- A staging site behaves like a normal site for read endpoints — point other `/xcloud-*` commands at its UUID to inspect it.
