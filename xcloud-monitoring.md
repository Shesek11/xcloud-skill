---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Show CPU/RAM/disk and HTTP performance stats for an xCloud server or site. Use to check resource usage, find slow sites, or verify a server is healthy.
---

## Context

- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`
- Site UUID:   !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Monitoring Stats

### Step 1: Resolve UUIDs

Read both UUIDs from `.xcloud/context.md`. If missing, run `/xcloud-init`.

### Step 2: Pick scope

Use AskUserQuestion:
- **Both** (default) — server + site
- **Server only** — CPU / RAM / disk / load
- **Site only** — request count / response times / bandwidth

### Step 3: Fetch

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"
H_ACC="Accept: application/json"
BASE="https://app.xcloud.host/api/v1"

# Server stats — requires read:servers
curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/servers/$SERVER_UUID/monitoring"

# Site stats — requires read:sites
curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/monitoring"
```

### Step 4: Render

The two endpoints have **different shapes**:
- **Server** returns a single snapshot: `{cpu: {...}, memory: {...}, disk: [...], recorded_at}`. All numeric values are **strings** — parse with care.
- **Site** returns a **time-series array** of `{ram_usage, cpu_usage, disk_usage, time_at}` (numbers). `time_at` is `"HH:MM AM/PM"` (no date). Aggregate (current = last point; avg / max across the array).

**Server snapshot:**

```
─── Server: [name]   (recorded_at [recorded_at]) ───
CPU:     [cpu.usedPercent]%   load 1/5/15m: [average_1_minute] [average_5_minutes] [average_15_minutes]
Cores:   [cpu.cores] ([cpu.threads] threads)
Uptime:  [cpu.uptime]
RAM:     [memory.used]/[memory.total] MB   ([memory.percent]%)   available [memory.available] MB
Swap:    [memory.swap_used]/[memory.swap_total] MB
Disk:
  [for each entry in disk[]:]
   [mountPoint]   [used]/[total]   ([usedPercent])
```

**Site time-series (aggregate over the returned points):**

```
─── Site: [domain]   ([N] data points) ───
Current:  RAM [last.ram_usage]%   CPU [last.cpu_usage]%   Disk [last.disk_usage]%
Average:  RAM [avg]%              CPU [avg]%              Disk [avg]%
Peak:     RAM [max]% @ [time_at]  CPU [max]% @ [time_at]
```

### Step 5: Flag issues

- Server `cpu.usedPercent` > 80 → suggest `/xcloud-ssh` to investigate processes.
- Any disk `usedPercent` > 90 → urgent, suggest cleanup.
- `memory.percent` > 90 → may explain OOM-killed deploys.
- `swap_used` ≈ `swap_total` (swap fully consumed) → server is paging heavily; very slow.
- Site CPU/RAM peaks consistently > 80 → app may need tuning or larger instance.

### Step 6: Suggest next steps

Based on flags:
- High disk → `/xcloud-ssh` then `du -sh ~/* | sort -h`
- High site response time → check PM2 (Node.js) or PHP-FPM workers via SSH
- Recurring spikes → consider scaling up via xCloud panel

## Important Notes

- The xCloud monitoring API does NOT expose request count, response time, bandwidth, or HTTP error rate. For app-level performance, use `/xcloud-ssh` (PM2 logs, Nginx access log) or external monitoring (e.g. Cloudflare analytics).
- Server stats are **string-typed** (`"100"`, `"57%"`). Strip `%`, cast to float as needed.
- Site stats are float points indexed by `time_at` (HH:MM AM/PM, no date). The series resets — don't assume points cross midnight cleanly.
- Stats are typically delayed 1–5 minutes. For real-time, SSH and run `htop` / `pm2 monit`.
