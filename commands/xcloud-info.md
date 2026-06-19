---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Show full info for an xCloud site — combines site detail, status, SSL, domain DNS check, git config, and SSH config in one view.
---

## Context

- Site UUID from context: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Site Info Aggregate

### Step 1: Resolve site UUID

- Read `.xcloud/context.md` to find `Site UUID:`. If not present, this project may need re-init or migration. Run `/xcloud-init` then return.
- Optional: user may pass a different UUID inline. Use AskUserQuestion if ambiguous.

### Step 2: Load token

Standard token loading from `~/.xcloud/token`. See `references/api-reference.md`.

### Step 3: Fetch in parallel

Make 5 calls and combine:

```bash
SITE=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID")
STATUS=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/status")
SSL=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/ssl")
DOMAIN=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/domain")
GIT=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/git")
SSH=$(curl -sS -H "$H_AUTH" -H "$H_ACC" "$BASE/sites/$SITE_UUID/ssh")
```

If any 403 → token lacks `read:sites` scope. Stop and instruct user to recreate the token.

### Step 4: Render unified view

The xCloud API returns provisioning status, not live HTTP. For a real HTTP check, fall back to a `curl -sI` against the public domain.

```bash
HTTP_CODE=$(curl -sI -o /dev/null -w "%{http_code}" "https://[domain_name]" --max-time 10)
```

Then render:

```
─── [domain_name] ───
Type:        [type]                  PHP: [php_version]
Provisioned: [is_provisioned]        Server: [server_uuid (short)]
Created:     [created_at]

▼ Live HTTP (independent check)
   HTTP [code]   ([up/down])

▼ SSL
   Provider:    [provider]                   (xcloud / letsencrypt / cloudflare)
   Status:      [status]                     (obtained / pending / failed)
   Expires:     [expires_at]                 ([N] days from now)
   Hostnames:   [hostnames or "n/a"]

▼ Domain
   Primary:     [primary_domain]
   Environment: [environment]                (production / staging)
   Additional:  [count] domain(s)             (some may be redirects)
   Cloudflare:  [active/inactive]

▼ Git
   Repo:        [git_repository]
   Branch:      [git_branch]
   Push-Deploy: [enable_push_deploy]
   Run script:  [run_after_deployment]
   .env file:   [present/missing]            (do NOT print contents — sensitive)

▼ SSH/SFTP
   User:        [site_user]
   Auth mode:   [authentication_mode]        (public_key / password)
   Keys:        [list each as "name (fingerprint)"]
```

### Step 5: Flag issues

- SSL `status` ≠ `obtained` → cert problem
- SSL `expires_at` in < 14 days → warn (more severe < 7 days)
- `enable_push_deploy: false` → mention so the user knows their pushes won't auto-deploy
- Live HTTP code ≠ 200 → warn
- `is_provisioned: false` → site not fully provisioned yet

### Step 6: Optional — write to context

Offer (via AskUserQuestion) to update the `## Server` and `## Git` sections of `.xcloud/context.md` with the latest values. Useful if anything was changed via the xCloud panel since the last init.

## Important Notes

- This is read-only. No state changes.
- The xCloud `/sites/{uuid}/status` endpoint returns **provisioning** state, not live HTTP. For "is the site actually responding", do an independent `curl -sI https://[domain]`.
- The SSL response has no `auto_renew` field. xCloud handles renewal internally — check `/xcloud-events` for renewal events if you need to verify.
- For deeper digging, point the user at `/xcloud-events`, `/xcloud-deploy-log`, `/xcloud-monitoring`.
- The `.env` content is NOT shown here — for that, read `git.env_file_content` directly via curl (and treat it as sensitive).
