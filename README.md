# xCloud Hosting Plugin

Manage xCloud hosting projects from Claude Code. Combines the **xCloud Public API** (auto-discovery, deploy verification, backups, monitoring, security scanning, WordPress management, and more) with **SSH / git / MariaDB** for everything the API doesn't cover.

> The xCloud Public API is now **v1.0.0** (out of beta) with **111 operations** across 11 categories. The plugin tracks the live spec snapshot in `references/openapi.yaml` (captured 2026-06-18) and pins to documented response fields where possible. See `CHANGELOG.md` for what changed.

---

## What's included

### Read / inspect (API-driven)

| Command | Purpose |
|---------|---------|
| `/xcloud` | Hub — shows status and routes to actions |
| `/xcloud-list` | List all servers and sites in your team |
| `/xcloud-info` | Full site detail: SSL, domain, git, SSH, status |
| `/xcloud-status` | Local + GitHub + xCloud API + (optional) SSH sync |
| `/xcloud-events` | Recent activity timeline (deploys, backups, config) |
| `/xcloud-deploy-log` | Deployment outputs + suggested fixes for common failures |
| `/xcloud-ssl` | SSL certificates: list, detail, status, **create, renew, delete** |
| `/xcloud-monitoring` | Server (CPU/RAM/disk) and site stats + history |

### Operate (API-driven, write)

| Command | Purpose |
|---------|---------|
| `/xcloud-backup` | Backups: list, trigger, **settings, status, count** |
| `/xcloud-cache-purge` | Flush full-page cache (or all cache layers) |
| `/xcloud-deploy` | Trigger a git deployment via API + view/update deploy settings |
| `/xcloud-reboot` | Reboot server (dual confirmation — affects ALL sites) |

### Manage / advanced (API-driven) — new

| Command | Purpose |
|---------|---------|
| `/xcloud-security` | Vulnerability scans, firewall rules, fail2ban (ban/unban IPs) |
| `/xcloud-wp` | WordPress: plugins/themes/updates/health, activate, update, magic-login, WP_DEBUG |
| `/xcloud-cron` | Cron jobs (site + server): list, create, update, delete, run, view output |
| `/xcloud-php` | PHP versions: install, uninstall, set default, OPcache, patch |
| `/xcloud-services` | Server services (restart/disable) + supervisor processes |
| `/xcloud-pagespeed` | PageSpeed Insights: latest snapshot, history, trigger scan |
| `/xcloud-snapshots` | List site/server snapshots |
| `/xcloud-staging` | List staging environments for a site |
| `/xcloud-redirects` | Redirections, web-rules (headers), IP access rules |

### Local + remote (SSH / git / MariaDB)

| Command | Purpose |
|---------|---------|
| `/xcloud-init` | Initialize a project — auto-discovers everything via API |
| `/xcloud-ssh` | Interactive shell, run commands, view logs, restart services |
| `/xcloud-db` | MariaDB queries via direct or tunneled connection |
| `/xcloud-pull` | git pull with conflict / stash handling |
| `/xcloud-push` | git commit + push + verify deploy via API events |

---

## Setup

### 1. Get an API token

See `references/token-setup.md`.

```bash
mkdir -p ~/.xcloud
printf '%s' 'YOUR_TOKEN_HERE' > ~/.xcloud/token
chmod 600 ~/.xcloud/token
```

The plugin reads `~/.xcloud/token` (or `$XCLOUD_API_TOKEN` env var). Token format looks like `34|...`.

Recommended scopes: `read:servers`, `write:servers`, `read:sites`, `write:sites`. Or `*` for full access.

### 2. Initialize a project

In any local directory (existing or new):

```bash
/xcloud-init
```

This will:
- List your sites via the API
- Let you pick which one to work on
- Auto-fill server, SSH, git, and DB info
- Clone the repo if needed
- Create `.xcloud/context.md`

If the directory already has a legacy `.xcloud/context.md` (from before the API), `/xcloud-init` runs in **migration mode** — it adds Site UUID + Server UUID without losing your `## Notes` or `## Development Log`.

### 3. Use the project

```bash
/xcloud         # hub
/xcloud-status  # full sync check
/xcloud-info    # site detail
/xcloud-push    # commit + push + verify
```

---

## Architecture

### Two stores

- **`~/.xcloud/token`** — global API token. One token works for all your projects.
- **`.xcloud/context.md`** — per-project metadata: Site UUID, Server UUID, paths, branch, DB info. Add to `.gitignore`.

### Token & API patterns

Every API command loads the token via this idiom:

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
```

See `references/api-reference.md` for the full pattern, including helpers, response envelope, status codes, and pagination.

### What the API can NOT do (still needs SSH)

- Run arbitrary shell commands
- Direct database queries — the `GET /servers/{uuid}/databases` endpoint was **removed**; use mysql via SSH tunnel
- Direct file edits on the server
- Create a server (no provisioning endpoint despite changelog hints)
- Create non-WordPress sites (only WordPress creation is exposed in the public API)

The `/xcloud-ssh`, `/xcloud-db`, and to some extent `/xcloud-pull` / `/xcloud-push` commands cover these.

---

## Security

- Passwords are NEVER stored.
- API token is at `~/.xcloud/token` with `chmod 600`. Outside any project tree.
- `.xcloud/context.md` does not contain DB password or API token. It does store SSH user, server IP, repo URL, branch — all non-secret identifiers.
- `git.env_file_content` is exposed by the API but the plugin never copies it into context.md. The DB password lives only on the server's `.env` file.

---

## Common workflows

### Push and verify

```bash
/xcloud-push    # commits, pushes, watches xCloud events for the new commit hash
```

If the deploy fails, the command surfaces the deploy log automatically.

### Daily check

```bash
/xcloud-status  # standard scope
```

This hits 4–5 API endpoints in parallel and shows: local repo state, GitHub remote, last deploy status, last backup, SSL expiry, live HTTP code. ~2–3 seconds.

### Before risky changes

```bash
/xcloud-backup  # → "Trigger backup"
```

Then make your DB migration / plugin update / etc.

### After a content change that's not showing up

```bash
/xcloud-cache-purge
```

### Audit your fleet

```bash
/xcloud-list     # see all sites
/xcloud-monitoring  # server + site stats
/xcloud-ssl      # cert health for current site (loop manually for fleet)
```

---

## Troubleshooting

### `401 Unauthorized` on every API call
- Check `~/.xcloud/token` exists and has no trailing newline (`printf` instead of `echo`).
- Verify with: `curl -H "Authorization: Bearer $(cat ~/.xcloud/token)" https://app.xcloud.host/api/v1/user`

### `403 Forbidden` on a specific endpoint
- Token lacks the required scope. Recreate at `https://app.xcloud.host/user/api-tokens` with the right scope.

### `/xcloud-init` finds zero sites
- Wrong team selected at xCloud panel. Switch teams in the panel and create a token there.

### Auto-deploy not triggering after push
- `/xcloud-info` → check `Push-Deploy: enabled`
- Check GitHub webhook deliveries in the repo settings
- Verify the deploy script runs: `/xcloud-deploy-log`

### `/xcloud-push` says "no deploy appeared"
- Push-deploy may be off; toggle in xCloud panel.
- GitHub webhook may be blocked; check `https://github.com/[user]/[repo]/settings/hooks` deliveries.
- The first push of a brand-new branch may not trigger; force one more push.

### Site type is `nodejs` but the API spec only lists `wordpress|custom-php|oneclick|static`
- Known beta inconsistency. The plugin handles `nodejs` and any other type as opaque strings.
- Node.js sites get all read endpoints (events, deploy logs, monitoring, etc.). Only **creation** is WordPress-only.

---

## Files

```
xcloud/
├── .claude-plugin/
│   └── plugin.json
├── README.md
├── CHANGELOG.md
├── commands/
│   ├── xcloud.md            # hub
│   ├── xcloud-init.md       # API-driven init + migration
│   ├── xcloud-list.md
│   ├── xcloud-info.md
│   ├── xcloud-status.md     # enriched with API
│   ├── xcloud-events.md
│   ├── xcloud-deploy-log.md
│   ├── xcloud-monitoring.md
│   ├── xcloud-backup.md     # list/trigger + settings/status/count
│   ├── xcloud-cache-purge.md
│   ├── xcloud-ssl.md        # full SSL cert resource (create/renew/delete)
│   ├── xcloud-reboot.md     # destructive
│   ├── xcloud-deploy.md     # NEW — trigger git deploy via API
│   ├── xcloud-security.md   # NEW — vulnerabilities + firewall + fail2ban
│   ├── xcloud-wp.md         # NEW — WordPress management + magic-login
│   ├── xcloud-cron.md       # NEW — cron jobs (site + server)
│   ├── xcloud-php.md        # NEW — PHP version management
│   ├── xcloud-services.md   # NEW — services + supervisor processes
│   ├── xcloud-pagespeed.md  # NEW — PageSpeed Insights
│   ├── xcloud-snapshots.md  # NEW — site/server snapshots
│   ├── xcloud-staging.md    # NEW — staging environments
│   ├── xcloud-redirects.md  # NEW — redirections / web-rules / IP access
│   ├── xcloud-ssh.md        # auto-loads from API
│   ├── xcloud-db.md         # mysql via tunnel
│   ├── xcloud-pull.md
│   └── xcloud-push.md       # verifies deploy via API
└── references/
    ├── token-setup.md
    ├── api-reference.md     # canonical 111-op catalog
    ├── openapi.yaml         # full spec snapshot (v1.0.0)
    └── ssh-key-setup.md
```

---

## Author

Custom plugin for xCloud hosting.
