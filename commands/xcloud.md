---
allowed-tools: Bash(git:*), Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Read, Write, Glob, AskUserQuestion
description: Main entry point for xCloud project management. Shows current project status and routes to API-driven operations (info, events, deploy logs, backups, cache purge, etc.) or SSH/git operations.
---

## Context

- Current directory: !`pwd`
- xCloud context: !`test -f .xcloud/context.md && echo "Yes — initialized" || echo "No — run /xcloud-init first"`
- Token file: !`test -f ~/.xcloud/token && echo "OK" || echo "MISSING — see plugins/xcloud/references/token-setup.md"`
- Git status: !`git status --short 2>/dev/null | head -5 || echo "Not a git repository"`

## xCloud Project Hub

### Step 1: Check token & context

- Token: read from `~/.xcloud/token` or `$XCLOUD_API_TOKEN`. If missing, point at `references/token-setup.md` and offer to walk through it.
- Context: if `.xcloud/context.md` is missing, suggest `/xcloud-init`. If present, read its Quick Reference (Site UUID, Server UUID, Domain, Type).

### Step 2: Show one-line summary

If both token and context are present, fetch lightweight live data (single API call):

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
curl -sS \
  -H "Authorization: Bearer $XCLOUD_TOKEN" \
  -H "Accept: application/json" \
  "https://app.xcloud.host/api/v1/sites/$SITE_UUID/status"
```

Display:

```
xCloud project: [domain_name]   ([type])
Server:         [server_name] ([ip])
HTTP:           [code, up/down]
Local changes:  [X uncommitted]
Branch:         [branch]
```

### Step 3: Pick action

Use AskUserQuestion. Group by purpose:

**Read / inspect:**
- **Status** (/xcloud-status) — local, GitHub, xCloud API, optional SSH
- **Info** (/xcloud-info) — full site detail (SSL, domain, git, SSH config)
- **List** (/xcloud-list) — all servers and sites in your team
- **Events** (/xcloud-events) — recent activity timeline
- **Deploy log** (/xcloud-deploy-log) — last deploy outputs + suggested fixes
- **SSL** (/xcloud-ssl) — certs: view/list/status/create/renew/delete
- **Monitoring** (/xcloud-monitoring) — server + site stats + history
- **PageSpeed** (/xcloud-pagespeed) — performance scores + scan
- **Redirects** (/xcloud-redirects) — redirections, headers, IP access
- **Snapshots** (/xcloud-snapshots) — restore points · **Staging** (/xcloud-staging)

**Operate (write):**
- **Backup** (/xcloud-backup) — list/trigger + status/settings/count
- **Deploy** (/xcloud-deploy) — trigger git deploy via API + settings
- **Purge cache** (/xcloud-cache-purge) — flush full-page or all caches
- **Push** (/xcloud-push) — commit, push, verify deploy via API
- **Pull** (/xcloud-pull) — git pull
- **SSH** (/xcloud-ssh) — interactive shell, run command, restart services
- **Database** (/xcloud-db) — mysql via tunnel

**Manage / advanced:**
- **Security** (/xcloud-security) — vulnerabilities, firewall, fail2ban
- **WordPress** (/xcloud-wp) — plugins/themes/updates, magic-login, WP_DEBUG
- **Cron** (/xcloud-cron) — cron jobs (site + server)
- **PHP** (/xcloud-php) — PHP versions (server-wide)
- **Services** (/xcloud-services) — restart/disable services, supervisor
- **Reboot server** (/xcloud-reboot) — DESTRUCTIVE / SHARED — affects all sites on server
- **Re-init** (/xcloud-init) — re-discover from API (also migrates legacy context)
- **View context** — print `.xcloud/context.md`
- **Add note** — append a note to `## Notes` section

### Step 4: Route

Based on selection, instruct the user to run the chosen `/xcloud-*` command, OR perform inline:
- View context — print the file
- Add note — prompt for text, append under `## Notes` with `### [DATE] - User Note`

## Important Notes

- The plugin uses two stores: `~/.xcloud/token` (global, for the API) and `.xcloud/context.md` (per-project, with UUIDs and metadata).
- `/xcloud-init` is non-destructive: re-running it migrates legacy context files without losing your `## Notes` and `## Development Log`.
- Most commands are read-only and safe. The destructive ones (`/xcloud-reboot`, `/xcloud-backup`, `/xcloud-push` to deploy branch, `/xcloud-cache-purge`) all require explicit confirmation.
