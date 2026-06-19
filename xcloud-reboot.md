---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Reboot the xCloud server hosting the current site. Use as a last resort when a service won't restart, or after kernel updates. Affects ALL sites on the server.
---

## Context

- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`
- Server name: !`test -f .xcloud/context.md && grep -E '\*\*Name:' .xcloud/context.md | head -1`

## Reboot Server

> **DESTRUCTIVE / SHARED**: A server reboot affects every site hosted on that server, not just the current project. Confirm with extreme care.

### Step 1: Resolve server UUID

Read `.xcloud/context.md` → `Server UUID`. Run `/xcloud-init` if missing.

### Step 2: Show what will be rebooted

Fetch the server's site list so the user sees impact:

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/servers/$SERVER_UUID/sites?per_page=100"
```

Show:

```
Server: [server_name] ([ip_address])
This reboot will briefly take DOWN the following sites:

  • exams.shahaf.org.il
  • jun-juhuri.com
  • [...]

Estimated downtime: 1–3 minutes.
```

### Step 3: Double confirmation

Two prompts. Both must pass.

**Prompt 1 (AskUserQuestion):**
- "Yes, reboot now"
- "Cancel"

If user picks "Yes, reboot now":

**Prompt 2 (free-form):** ask the user to type the server name **exactly** (e.g. `Apps`) to confirm. Compare strings; abort on mismatch.

This dual-confirm is intentional — single-click reboots are too easy to fire accidentally.

### Step 4: Send reboot

```bash
curl -sS -w "\n__HTTP__:%{http_code}" \
  -X POST \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/servers/$SERVER_UUID/reboot"
```

Expected: HTTP **202**.

If 403 → token lacks `write:servers`. Stop and inform user.

### Step 5: Watch

Poll `/servers/{uuid}/tasks` and/or pings to the IP every 10s until the server is back up. Show:

```
[00:00] Reboot queued.
[00:05] Server going down...
[00:30] No ping response (expected).
[01:15] Server responding.
[01:20] HTTP 200 from [first-site].

✓ Reboot complete in 1m 20s.
```

### Step 6: Post-reboot checks

After the server is back, automatically run:

1. `GET /servers/{uuid}` → confirm status `provisioned`.
2. For each site listed in Step 2, `GET /sites/{uuid}/status` → flag any non-200.

### Step 7: Update context

Append to `## Recent Deployments` (or a `## Server Operations` section):

```markdown
### [DATE] [TIME] - Server Reboot
- Server: [name]
- Affected sites: [count]
- Downtime: [duration]
- Result: [all sites back up / X sites failing]
```

## Important Notes

- This is the only **shared-blast-radius** command in the plugin. Always run the dual confirmation.
- For Node.js apps without `@reboot` cron persistence, PM2 processes will NOT come back automatically — see `xcloud-nodejs-persistence` skill before rebooting.
- If your goal is to restart just one app/service, prefer `/xcloud-ssh` → "Restart Services" instead.
