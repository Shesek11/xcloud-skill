---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Manage PHP versions on an xCloud server — list installed/available, install or uninstall a version, set the default, toggle OPcache, and apply security patches. Server-scoped, so it affects every site on the server.
---

## Context

- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`

## xCloud PHP Versions

See `plugins/xcloud/references/api-reference.md` for the auth helper.

### Step 1: Resolve server UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:
- **List installed** (default) — versions on the server + which is default, OPcache state
- **List available** — versions that can be installed + patch availability
- **Patch info** — pending PHP patches
- **Install** — add a PHP version (async)
- **Uninstall** — remove a PHP version (destructive)
- **Set default** — make a version the server default
- **Toggle OPcache** — enable/disable OPcache for a version
- **Apply patch** — patch a version (async)

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"

curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/php-versions"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/php-versions/available"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/php-versions/patch-info"

# Install / uninstall (php_version in the body for both)
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"php_version":"8.3"}' "$B/servers/$SERVER_UUID/php-versions"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"php_version":"8.1"}' "$B/servers/$SERVER_UUID/php-versions"

# Version-specific actions ({version} in the path, e.g. 8.3)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/php-versions/8.3/default"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"enabled":true}' "$B/servers/$SERVER_UUID/php-versions/8.3/opcache"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/php-versions/8.3/patch"
```

### Step 4: Render

- **Installed** — table: version, is_default, opcache_enabled, patch_available.
- **Available** — versions installable + whether a patch exists.

### Step 5: Confirm writes

Confirm before **Install / Uninstall / Set default / Patch / OPcache toggle**, and remind the user this is **server-wide**:
- **Uninstalling** a PHP version breaks any site using it — verify no site depends on it first.
- **Set default** changes the version new sites (and CLI) use.
- **Install** and **Patch** are **async** — poll `/servers/{uuid}/tasks` to confirm completion.

## Important Notes

- PHP management is server-scoped — changes hit every site on the server.
- A Node.js-only / Docker server may report no PHP versions installed (empty list) — that's normal.
- After changing the default or applying a patch, consider restarting PHP-FPM via `/xcloud-services`.
