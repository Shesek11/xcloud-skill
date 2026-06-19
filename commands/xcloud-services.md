---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Inspect and control xCloud server services (nginx, mysql, php-fpm, docker, redis, …) and supervisor-managed processes. List status, restart a service, disable a service, or view supervisor processes.
---

## Context

- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`

## xCloud Server Services

See `plugins/xcloud/references/api-reference.md` for the auth helper.

### Step 1: Resolve server UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:
- **List services** (default) — name, status, version, auto-healing
- **Restart a service**
- **Disable a service**
- **Supervisor processes** — list supervisor-managed workers

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"

curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/services"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/supervisor-processes?per_page=50"

# Restart / disable — service name from the list; version optional (e.g. PHP-FPM version)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"service":"nginx"}'            "$B/servers/$SERVER_UUID/services/restart"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"service":"php-fpm","version":"8.3"}' "$B/servers/$SERVER_UUID/services/restart"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"service":"redis"}'            "$B/servers/$SERVER_UUID/services/disable"
```

### Step 4: Render & flag

- Table: `service | label | status | version | required | auto-healing`.
- Flag any service with `status: failed` or `stopped` that is `is_required: true` — that likely explains site outages.
- A non-required service showing `failed` (e.g. `docker` on a non-Docker server) is usually harmless — note it but don't alarm.

### Step 5: Confirm writes

Confirm before **Restart / Disable**. Both affect the **whole server**:
- Restarting `nginx`/`php-fpm` causes a brief blip for all sites.
- **Disabling a required service takes sites down** — only disable if the user is certain (e.g. turning off an unused `redis`).

## Important Notes

- Server-scoped: every site on the server is affected.
- For deeper control (start/stop a single PM2 app, tail logs), use `/xcloud-ssh`.
- After PHP changes via `/xcloud-php`, restart `php-fpm` here to apply.
