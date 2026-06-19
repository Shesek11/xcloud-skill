---
allowed-tools: Bash(git clone:*), Bash(mkdir:*), Bash(ls:*), Bash(test:*), Bash(curl:*), Bash(cat:*), Bash(grep:*), Bash(sed:*), Bash(awk:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Write, Glob, Grep, AskUserQuestion
description: Initialize an xCloud project using the API for auto-discovery. Pulls server, SSH, git, and DB info automatically. Also migrates older `.xcloud/context.md` files to the new format.
---

## Context

- Current directory: !`pwd`
- Existing context: !`test -f .xcloud/context.md && echo "FOUND — will migrate" || echo "absent — fresh init"`
- Token file: !`test -f ~/.xcloud/token && echo "OK" || echo "MISSING — see references/token-setup.md"`
- Token env: !`test -n "$XCLOUD_API_TOKEN" && echo "OK (env)" || echo "absent"`

## Initialize / Migrate xCloud Project

This command auto-discovers everything from the xCloud API. The user only picks **which site** they want to work on locally.

### Step 0: Check token availability

Read `references/token-setup.md` if either of the following is true:
- The token file `~/.xcloud/token` does not exist AND `$XCLOUD_API_TOKEN` is empty.
- A test call to `/user` returns 401.

Test call:

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(test -f ~/.xcloud/token && tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  https://app.xcloud.host/api/v1/user
```

If not 200, abort with the token-setup instructions. Do not continue without a working token.

### Step 1: Detect mode (fresh init vs migration)

- If `.xcloud/context.md` already exists in the current dir, this is a **migration**. Read the existing file to extract `domain_name` (look at the `Server:` or `**GitHub:**` lines, or any line that mentions a domain).
- Otherwise, this is a **fresh init**.

### Step 2a: Migration path

If migrating:

1. Parse the existing `context.md` for clues — the project's domain or GitHub repo URL.
2. Call the API to find the matching site:

   ```bash
   curl -sS -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" \
     "https://app.xcloud.host/api/v1/sites?per_page=100" \
     | python -c "
   import sys, json
   target = '$DOMAIN'  # filled in from context parse
   data = json.load(sys.stdin)['data']['items']
   match = [s for s in data if target in s['domain_name']]
   for s in match:
       print(s['uuid'], s['domain_name'], s['type'], s['server_uuid'])
   "
   ```

3. If exactly one match → proceed with that site UUID. If zero → fall through to fresh init flow (offer the user a list). If multiple → use AskUserQuestion to pick.
4. Skip to Step 5 (build context).

### Step 2b: Fresh init path

Use AskUserQuestion or paginated listing:

1. Call `GET /sites?per_page=100` and present the user with a list of sites: `domain_name (type, server_name)`.
2. User picks one. Capture its `uuid`, `server_uuid`, `domain_name`, `type`.

Skip to Step 3.

### Step 3: Auto-discover from API

Once we have a `SITE_UUID` and `SERVER_UUID`, fetch in parallel:

```bash
SITE_JSON=$(curl -sS -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" "$XCLOUD_BASE/sites/$SITE_UUID")
SSH_JSON=$(curl -sS  -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" "$XCLOUD_BASE/sites/$SITE_UUID/ssh")
GIT_JSON=$(curl -sS  -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" "$XCLOUD_BASE/sites/$SITE_UUID/git")
SRV_JSON=$(curl -sS  -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" "$XCLOUD_BASE/servers/$SERVER_UUID")
```

Extract these fields (use `python -c "import sys,json; print(json.loads(sys.stdin.read())['data']['...'])"` or `jq`):

| From | Field | Used for |
|------|-------|----------|
| site | `domain_name` | Domain |
| site | `type` | Site type (nodejs/wordpress/...) |
| site | `php_version` | PHP version |
| ssh  | `site_user` | SSH username |
| ssh  | `authentication_mode` | Key vs password |
| git  | `git_repository` | GitHub URL |
| git  | `git_branch` | Default branch |
| git  | `enable_push_deploy` | Auto-deploy yes/no |
| git  | `env_file_content` | **Optional** parse for DB creds (do not save raw — see Step 4) |
| srv  | `name` | Server label |
| srv  | `ip_address` | Server IP |
| srv  | `location` | Server region |
| srv  | `provider` | Hosting provider |

### Step 4: Optional DB info (parse, never store passwords)

If `git.env_file_content` is non-null, parse out:

- `DB_HOST`
- `DB_PORT`
- `DB_DATABASE`
- `DB_USERNAME`

**Do not** copy `DB_PASSWORD` into context.md. Only the non-secret fields go into context. The user will type the password into the mysql client each time (handled by `/xcloud-db`).

### Step 5: Build / update `.xcloud/context.md`

Use these defaults when fields are absent:
- SSH port: `22`
- Remote path: `~/[domain_name]` (xCloud convention; can be overridden in Step 6)

Write the file with the following structure (overwrite for fresh init; merge-update for migration — keep `## Notes` and `## Development Log` from the old file):

```markdown
# Project: [domain_name]

> Auto-discovered from xCloud API. Site UUID and Server UUID are the source of truth.
> All `/xcloud-*` commands rely on these.

## Quick Reference
- **Site UUID:**   [site_uuid]
- **Server UUID:** [server_uuid]
- **Domain:**      [domain_name]
- **Type:**        [type] (e.g. nodejs / wordpress / static)
- **PHP Version:** [php_version]

## Server
- **Name:**     [server_name]
- **IP:**       [ip_address]
- **Location:** [location]
- **Provider:** [provider]
- **SSH:**      [site_user]@[ip_address]:22
- **Auth Mode:**[authentication_mode]
- **Remote Path:** ~/[domain_name]   <!-- override if non-standard -->

## Git
- **Repo:**          [git_repository]
- **Branch:**        [git_branch]
- **Push-Deploy:**   [enable_push_deploy]
- **Local Path:**    [user-chosen path]

## Database
<!-- Auto-populated from env file if present. Password is NEVER stored. -->
- **Host:**     [DB_HOST or 'localhost']
- **Port:**     [DB_PORT or 3306]
- **Database:** [DB_DATABASE or unknown]
- **User:**     [DB_USERNAME or unknown]

## Connection Commands
```bash
# SSH (uses port 22 by default)
ssh [site_user]@[ip_address]

# DB (password prompt by mysql client)
mysql -h [DB_HOST] -P [DB_PORT] -u [DB_USERNAME] --password [DB_DATABASE]
```

## Development Log
<!-- Newest entries on top -->

### [DATE] - Project initialized via API
- Auto-discovered from /sites/[site_uuid]
- Migration: [yes/no]

---

## Database Schema
<!-- Populated by /xcloud-db -->

## Recent Deployments
<!-- Populated by /xcloud-push and /xcloud-deploy-log -->

## Notes
<!-- Add free-form notes here. Preserved across re-init. -->
```

### Step 6: Clone or verify repo

- If `.git` does not exist in the local path: `git clone [git_repository] [local_path]`
- If it does: confirm the remote matches `git_repository`. If not, warn but do not change.

### Step 7: Confirm .gitignore

Ensure `.xcloud/` is in `.gitignore`. Append if missing:

```bash
grep -qxF '.xcloud/' .gitignore 2>/dev/null || echo '.xcloud/' >> .gitignore
```

### Step 8: Output summary

Show the user:

```
✓ xCloud project initialized: [domain_name]
  Site UUID:   [site_uuid]
  Server:      [server_name] ([ip_address])
  Type:        [type]
  Auto-deploy: [yes/no]

Next steps:
  /xcloud-list         — see all your sites
  /xcloud-info         — full status of this site
  /xcloud-status       — local + remote + server sync
  /xcloud-events       — recent activity timeline
  /xcloud-deploy-log   — last deployment output
```

## Important Notes

- **Never store secrets**: API token lives globally at `~/.xcloud/token`; SSH passwords prompted by `ssh`; DB passwords prompted by `mysql`.
- **Migration is non-destructive**: existing `## Notes` and `## Development Log` sections are preserved.
- **SSH port** is hardcoded to `22` because the xCloud API does not expose it. If your server uses a non-standard port, edit the `**SSH:**` line in context.md manually.
- **Remote path** defaults to `~/[domain_name]` (xCloud convention). Edit if your site is at a different path.
- If the site `type` is `nodejs`, set up PM2 persistence per `xcloud-nodejs-persistence` skill.
