# xCloud API Token Setup

The xCloud Public API requires a Sanctum Personal Access Token. This file documents how to obtain a token and store it locally so all `/xcloud-*` commands can use it without prompting.

## Step 1: Create a token in the xCloud panel

1. Log in to https://app.xcloud.host
2. Open **Account → API Tokens** (`https://app.xcloud.host/user/api-tokens`)
3. Click **Create New Token**
4. Pick the scopes you need (recommended for this plugin):
   - `read:servers` — list/inspect servers, monitoring, cron, php-versions, services, firewall, snapshots
   - `write:servers` — reboot, sudo users, create WordPress sites, cron, firewall, fail2ban, php-versions, services
   - `read:sites` — list/inspect sites, backups, SSL, deployment logs, SSH, vulnerabilities, WordPress, pagespeed
   - `write:sites` — backups, cache purge, SSH, git deploy, SSL create/renew, cron, WordPress actions, vulnerability scans
   - Or `*` for full access — **recommended** so all 111 operations work without scope surprises
5. Copy the token — **it is shown only once**

> The v1.0.0 OpenAPI spec declares only a bearer scheme and no longer lists scopes per endpoint, but scopes are still enforced at the token level. If any call returns **403**, the token is missing that resource's scope — recreate it with the right scope (or `*`).

Token format looks like: `34|CAXMex9zjFyZEayWtG78TueMgIDU8iQEXjSLhlqhb5573ac4`

## Step 2: Store the token locally

The plugin reads the token from `~/.xcloud/token` (or the `XCLOUD_API_TOKEN` env var as a fallback).

**On Windows (Git Bash / WSL):**

```bash
mkdir -p ~/.xcloud
printf '%s' 'YOUR_TOKEN_HERE' > ~/.xcloud/token
chmod 600 ~/.xcloud/token  # Best-effort on Windows; works on WSL/Git Bash
```

**On macOS / Linux:**

```bash
mkdir -p ~/.xcloud
printf '%s' 'YOUR_TOKEN_HERE' > ~/.xcloud/token
chmod 600 ~/.xcloud/token
```

> Use `printf` rather than `echo` to avoid a trailing newline in the token file. A trailing `\n` will make every API call return 401.

## Step 3: Verify it works

```bash
TOKEN=$(cat ~/.xcloud/token)
curl -s -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" \
  https://app.xcloud.host/api/v1/user
```

Expected response:

```json
{ "success": true, "message": "Success", "data": { "uuid": "...", "name": "...", "email": "...", ... } }
```

If you see `"success": false` or HTTP 401, the token is wrong. Recreate it.

## Rotating / revoking tokens

To revoke a token, either:

1. Open `https://app.xcloud.host/user/api-tokens` and delete it from the UI, or
2. Run `/xcloud-list` → tokens, or
3. Call the API directly:

```bash
curl -X DELETE -H "Authorization: Bearer $TOKEN" -H "Accept: application/json" \
  https://app.xcloud.host/api/v1/user/tokens/{tokenId}
```

After revoking, re-run Step 1–2 with a new token.

## Security notes

- The token file should be `chmod 600` (owner read/write only).
- Never commit `~/.xcloud/token` to git — it lives in your home folder, outside any project.
- Never paste the token into `.xcloud/context.md` (per-project) — that file may be tracked by git.
- If you suspect leakage, revoke immediately via the panel.
