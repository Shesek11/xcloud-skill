---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, AskUserQuestion
description: List all servers and sites accessible to the current xCloud team. Use to see what's available, find a site UUID, or check overall account state.
---

## Context

- Token file: !`test -f ~/.xcloud/token && echo "OK" || echo "MISSING"`

## List xCloud Servers and Sites

### Step 1: Load token

Read the token from `~/.xcloud/token` or `$XCLOUD_API_TOKEN`. Abort with link to `references/token-setup.md` if missing.

### Step 2: Ask scope

Use AskUserQuestion:
- **All** — both servers and sites (default)
- **Servers only**
- **Sites only**
- **One server's sites** — pick a server, list its sites

### Step 3: Fetch data

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
BASE="https://app.xcloud.host/api/v1"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"
H_ACC="Accept: application/json"

# Servers (small list, no pagination needed for typical accounts)
curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/servers?per_page=100"

# Sites (paginated; user may have many)
curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites?per_page=100"
```

For "One server's sites":

```bash
curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/servers/$SERVER_UUID/sites?per_page=100"
```

### Step 4: Display

**Servers table:**

| Name | Provider | Location | IP | Status | UUID (short) |
|------|----------|----------|-----|--------|--------------|

**Sites table:**

| Domain | Type | Server | Status | UUID (short) |
|--------|------|--------|--------|--------------|

Show full UUIDs only on request (they're long; truncate to first 8 chars by default).

If pagination has more pages (`pagination.last_page > current_page`), tell the user how many sites total and offer to fetch more.

### Step 5: Cross-reference current project

If `.xcloud/context.md` exists in the current dir, highlight the matching site row with `★`. This helps the user confirm the right project is selected.

## Important Notes

- This is a read-only command. No write scopes used.
- Output format: prefer markdown table for ≤ 30 rows; switch to plain text columns for more.
- For large accounts (100+ sites), suggest filtering by name/domain client-side rather than fetching all pages.
