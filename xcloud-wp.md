---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Manage a WordPress site on xCloud via the API — list plugins/themes/updates, check site health, activate or update items, toggle WP_DEBUG, and generate a one-time magic admin login URL. Use for WordPress maintenance without SSH or wp-admin.
---

## Context

- Site UUID:  !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`
- Site type:  !`test -f .xcloud/context.md && grep -E 'Type' .xcloud/context.md | head -1`

## xCloud WordPress Management

See `references/api-reference.md` for the auth helper and bodies.

### Step 1: Resolve site UUID + confirm it's WordPress

Read `.xcloud/context.md`. Run `/xcloud-init` if missing. If `Type` is not `wordpress`, warn: the WordPress endpoints return empty/null on non-WP sites — confirm the user still wants to proceed.

### Step 2: Pick action

Use AskUserQuestion:

**Inspect (read):**
- **Health** (default) — `GET /wordpress/status`
- **Updates** — core/plugin/theme update summary
- **Plugins** — list (filter active/inactive, search)
- **Themes** — list

**Act (write):**
- **Update items** — update plugins or themes (optionally all)
- **Activate items** — activate plugins/themes
- **Refresh** — re-scan installed items from the server
- **Toggle WP_DEBUG** — turn debug logging on/off
- **Magic login** — generate a one-time admin login URL

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"

# Read
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/wordpress/status"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/wordpress/updates"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/wordpress/plugins?status=all&per_page=100"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/wordpress/themes?status=all&per_page=100"

# Update (type: plugin|theme; omit slugs to update everything of that type)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"type":"plugin","slugs":["woocommerce"],"backup_before_update":true}' \
  "$B/sites/$SITE_UUID/wordpress/update"

# Activate
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"type":"plugin","slugs":["woocommerce"],"backup_before_action":true}' \
  "$B/sites/$SITE_UUID/wordpress/activate"

# Refresh inventory
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/wordpress/refresh"

# WP_DEBUG on/off
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"enabled":true}' "$B/sites/$SITE_UUID/wp-debug"

# Magic login (login_as optional — defaults to the site admin user)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"login_as":"admin"}' "$B/sites/$SITE_UUID/magic-login"
```

### Step 4: Render

- **Updates** — show counts of available core/plugin/theme updates; highlight security updates (`include_security_only=1` to filter).
- **Plugins/themes** — table of name, version, status (active/inactive), update-available. Use the response `summary` block for totals.
- **Health** — surface any failed checks from `wordpress/status`.

### Step 5: Confirm writes & handle results

- Confirm before **update/activate** — list exactly which slugs. Default `backup_before_update`/`backup_before_action` to **true** unless the user opts out.
- Updates/activations may run **async** — after a 202, suggest re-running "Updates" or "Plugins" to confirm.
- **Magic login URL is a live admin credential.** Print it once, tell the user it is one-time/short-lived, and do NOT write it to `context.md`, logs, or anywhere persistent.

## Important Notes

- These endpoints only do something meaningful on `type: wordpress` sites.
- For anything not exposed here (wp-cli, direct DB), use `/xcloud-ssh` or `/xcloud-db`.
- `backup_before_update: true` is cheap insurance before bulk updates — prefer it.
