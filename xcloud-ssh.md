---
allowed-tools: Bash(ssh:*), Bash(ssh-keygen:*), Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, Write, AskUserQuestion
description: Connect to the xCloud server via SSH. Auto-loads SSH config (user, host, port, auth mode) from the API; falls back to context.md. Use for shells, command execution, log viewing, service restarts, and SSH key setup.
---

## Context

- Current directory: !`pwd`

## SSH Connection

### Step 1: Find context

Locate `.xcloud/context.md` (current dir or parents). If not found → run `/xcloud-init` first.

### Step 2: Resolve connection details

Pull `Site UUID` and `Server UUID` from context. If present, prefer **live API** (handles changes made via the xCloud panel):

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"
H_ACC="Accept: application/json"
BASE="https://app.xcloud.host/api/v1"

SSH_JSON=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/ssh")
SRV_JSON=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/servers/$SERVER_UUID")

# Extract:
# - SSH_USER from SSH_JSON.data.site_user
# - AUTH_MODE from SSH_JSON.data.authentication_mode (public_key / password)
# - HOST from SRV_JSON.data.ip_address
# - PORT defaults to 22 (xCloud convention; not exposed by API)
# - REMOTE_PATH defaults to ~/[domain_name]
```

If the API is unreachable, fall through to values from `.xcloud/context.md`.

### Step 3: Test SSH key auth

```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes -p 22 [SSH_USER]@[HOST] "echo SSH_KEY_OK" 2>/dev/null
```

- If `SSH_KEY_OK` → key auth works.
- Else → key auth not set up. Add "Setup SSH Key Auth" to the action menu.

### Step 4: Pick action

Use AskUserQuestion:
- **Setup SSH Key Auth** (only if key auth fails) — uses `references/ssh-key-setup.md`. After success, can also push the key into the xCloud-managed authorized list via the API (Step 5b below).
- **Interactive Shell** — `ssh [user]@[host]`
- **Run Command** — execute a single command (with `nohup` wrapper for long commands)
- **Check Server Status** — uptime, disk, memory, PM2, node procs
- **PM2 Diagnostics** — full PM2 inspection
- **View Logs** — PM2 / Nginx / app log
- **Restart Services** — PM2 reload, nginx restart, php-fpm restart

### Step 5a: Execute SSH (standard cases)

**Interactive Shell:**
```bash
ssh [SSH_USER]@[HOST]
```

**Run Command — short:**
```bash
ssh [SSH_USER]@[HOST] "[command]"
```

**Run Command — long-running (build, migration):**
```bash
ssh [SSH_USER]@[HOST] "nohup bash -c '[command]' > /tmp/nohup-output.log 2>&1 &"
ssh [SSH_USER]@[HOST] "tail -20 /tmp/nohup-output.log"
```

**Check Server Status:**
```bash
ssh [SSH_USER]@[HOST] "echo '=== UPTIME ===' && uptime && echo '=== DISK ===' && df -h ~/[domain] | tail -1 && echo '=== MEMORY ===' && free -m | grep Mem && echo '=== PM2 ===' && pm2 list 2>/dev/null && echo '=== NODE PROCESSES ===' && ps aux | grep -E 'node|pm2' | grep -v grep | head -10"
```

**PM2 Diagnostics:**
```bash
ssh [SSH_USER]@[HOST] "
echo '=== PM2 PROCESS LIST ===';
pm2 list 2>/dev/null;
echo '=== PM2 RECENT LOGS (last 30 lines) ===';
pm2 logs --nostream --lines 30 2>/dev/null;
echo '=== REBOOT PERSISTENCE ===';
crontab -l 2>/dev/null | grep '@reboot' || echo 'NO @reboot cron found — app will die on restart!';
echo '=== ECOSYSTEM CONFIGS ===';
ls -la ~/*/ecosystem.config.js ~/[domain]/ecosystem.config.* 2>/dev/null;
echo '=== PORT USAGE ===';
ss -tlnp 2>/dev/null | grep -E ':(3000|2998|8080)' || echo 'No common Node ports in use';
"
```

**View Logs:**
```bash
# PM2 errors:
ssh [SSH_USER]@[HOST] "pm2 logs --nostream --lines 50 2>/dev/null"

# Nginx error:
ssh [SSH_USER]@[HOST] "tail -100 ~/logs/error.log 2>/dev/null || tail -100 /var/log/nginx/error.log 2>/dev/null"

# Nginx access:
ssh [SSH_USER]@[HOST] "tail -50 ~/logs/access.log 2>/dev/null"
```

**Restart Services:**
```bash
ssh [SSH_USER]@[HOST] "pm2 reload [process-name] && pm2 save"
ssh [SSH_USER]@[HOST] "sudo service nginx restart"
ssh [SSH_USER]@[HOST] "sudo service php8.2-fpm restart"
```

### Step 5b: Setup SSH Key Auth (with API option)

Read `references/ssh-key-setup.md` for the manual steps (generate key, push to authorized_keys via password SSH).

**Bonus**: after a successful manual setup, optionally also register the public key with xCloud via the API so future provisioning includes it:

```bash
# Get current SSH config
CUR=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/ssh")

# Build new config: append our local public key to the existing list
PUB_KEY=$(cat ~/.ssh/id_ed25519.pub)
NEW_BODY=$(python -c "
import json, sys
cur = json.loads('''$CUR''')['data']
keys = [k['fingerprint'] for k in cur.get('ssh_keypairs', [])]  # existing fingerprints (read-only)
# The PUT API replaces the keys; we have to send ssh_public_keys as the full new list
print(json.dumps({
    'authentication_mode': 'public_key',
    'ssh_public_keys': ['$PUB_KEY']  # WARNING: replaces existing — confirm with user first
}))
")

# Confirm with user before sending PUT
curl -sS -X PUT \
  -H "$H_AUTH" -H "$H_ACC" -H "Content-Type: application/json" \
  -d "$NEW_BODY" \
  "$BASE/sites/$SITE_UUID/ssh"
```

⚠ The PUT replaces the entire key list. Always confirm with the user before sending — losing access mid-session would be bad.

### Step 6: Update context

Append to `## Development Log`:

```markdown
### [DATE] [TIME] - SSH Session
- Action: [what was done]
- Result: [outcome]
```

## Important Notes

- **API-first lookup**: connection details come from the API when possible (always-fresh), with `.xcloud/context.md` as a fallback for offline use.
- **SSH port** is hardcoded to 22 — the xCloud API does not expose a custom port. Override in context.md if your server is non-standard.
- **Long commands**: ALWAYS wrap with `nohup` for builds/migrations. SSH timeouts are the #1 cause of interrupted operations.
- **Passwords with special chars** (`!`, `$`, `#`): use a Python/Node script instead of inline bash to avoid escaping bugs.
- **PUT /sites/{uuid}/ssh replaces the full key list** — never call it without explicit user confirmation.
