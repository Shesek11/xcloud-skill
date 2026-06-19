# xCloud API Reference (for `/xcloud-*` commands)

This is the canonical reference all xcloud commands rely on. Read this once before issuing API calls. The full OpenAPI spec is alongside in `openapi.yaml`.

> **Status:** The xCloud Public API graduated from beta. Current spec is **v1.0.0** with **111 operations** across 11 categories. Snapshot captured 2026-06-18 from `https://app.xcloud.host/api/v1/docs`.

## Base URL & auth

```
Base URL: https://app.xcloud.host/api/v1
Auth:     Authorization: Bearer <token>
Accept:   application/json
```

Token is loaded from `~/.xcloud/token` (preferred) or `$XCLOUD_API_TOKEN`. See `token-setup.md`.

## Bash auth helper (paste this at the top of any command bash block)

```bash
# Load token: prefer env var, then file. Strip any trailing newline.
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(test -f ~/.xcloud/token && tr -d '\r\n' < ~/.xcloud/token)}"
if [ -z "$XCLOUD_TOKEN" ]; then
  echo "ERROR: No xCloud API token found. See plugins/xcloud/references/token-setup.md"
  exit 1
fi
XCLOUD_BASE="https://app.xcloud.host/api/v1"

# Helper: GET with JSON pretty-print and HTTP status check
xc_get() {
  local path="$1"
  curl -sS -w "\n__HTTP__:%{http_code}" \
    -H "Authorization: Bearer $XCLOUD_TOKEN" \
    -H "Accept: application/json" \
    "$XCLOUD_BASE$path"
}

# Helper: POST/PUT/DELETE with JSON body
xc_send() {
  local method="$1" path="$2" body="${3:-}"
  if [ -n "$body" ]; then
    curl -sS -X "$method" -w "\n__HTTP__:%{http_code}" \
      -H "Authorization: Bearer $XCLOUD_TOKEN" \
      -H "Accept: application/json" \
      -H "Content-Type: application/json" \
      -d "$body" \
      "$XCLOUD_BASE$path"
  else
    curl -sS -X "$method" -w "\n__HTTP__:%{http_code}" \
      -H "Authorization: Bearer $XCLOUD_TOKEN" \
      -H "Accept: application/json" \
      "$XCLOUD_BASE$path"
  fi
}
```

The trailing `__HTTP__:NNN` lets you check status without `-i` polluting the JSON. To strip it before parsing:

```bash
RESPONSE=$(xc_get "/sites")
HTTP_CODE=$(echo "$RESPONSE" | tail -n1 | cut -d: -f2)
JSON=$(echo "$RESPONSE" | sed '$d')
```

## Response envelope

Every response (success or failure) follows:

```json
{ "success": true, "message": "...", "data": { ... } }
```

For paginated lists:

```json
{ "success": true, "data": { "items": [...], "pagination": { "total": 72, "per_page": 15, "current_page": 1, "last_page": 5 } } }
```

Some list endpoints add a `summary` or `counts` block alongside `items` (e.g. vulnerabilities, WordPress plugins, web-rules, domains). Read it for at-a-glance totals instead of counting `items` yourself.

## HTTP status codes you'll see

| Code | Meaning | Action |
|------|---------|--------|
| 200 | OK | Use `data` |
| 201 | Created | Resource created — use `data` |
| 202 | Accepted | Async op queued (reboot, backup, scan, deploy, php install) — poll events/tasks to verify |
| 400 | Validation failed | Check `errors` field |
| 401 | Bad/missing token | Re-run `token-setup.md` |
| 403 | Insufficient scope / not your team | Recreate token with required scope |
| 404 | Not found / not in your team | Verify UUID and team |
| 422 | Validation / business logic failure | Read `message` + `errors` (e.g. bad `range` value) |
| 429 | Rate limited | Back off and retry |
| 5xx | Server error | Retry; if persists, report to xCloud support |

## Authentication scopes

The v1.0.0 OpenAPI spec declares a single security scheme (`bearerAuth`) and **no longer enumerates per-endpoint scopes**. Scopes still exist at **token creation time** in the panel. Practical guidance:

| Scope | Covers |
|-------|--------|
| `read:servers` | List/inspect servers, monitoring, cron, php-versions, services, firewall, snapshots, sudo users |
| `write:servers` | Reboot, create WP sites, manage sudo users, cron, firewall, fail2ban, php-versions, services |
| `read:sites` | List/inspect sites, backups, SSL, domains, git, deploy logs, SSH, vulnerabilities, WordPress, pagespeed |
| `write:sites` | Backups, cache purge, SSH, git deploy, SSL create/renew, cron, WordPress actions, vulnerability scans |
| `*` | Full access — simplest for this plugin |

If any call returns **403**, the token is missing the scope for that resource — recreate it at `https://app.xcloud.host/user/api-tokens` (use `*` if unsure).

---

## Endpoint catalog (111 operations)

### Health & user (4)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | Liveness probe (no auth) |
| GET | `/user` | Current user profile |
| GET | `/user/tokens` | List your tokens |
| DELETE | `/user/tokens/{tokenUuid}` | Revoke a token (param is **`tokenUuid`**, renamed from `tokenId`) |

### Servers — read (18)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/servers` | Paginated list — `?search=&status=&page=&per_page=` |
| GET | `/servers/{uuid}` | Full server detail |
| GET | `/servers/{uuid}/sites` | Sites on a server — `?status=&type=` |
| GET | `/servers/{uuid}/monitoring` | CPU/RAM/disk snapshot (string values) |
| GET | `/servers/{uuid}/monitoring/history` | Time-series — `?range=24h\|7d` (default `7d`) |
| GET | `/servers/{uuid}/tasks` | Recent provisioner tasks |
| GET | `/servers/{uuid}/cron-jobs` | Server-level cron entries |
| GET | `/servers/{uuid}/cron-jobs/{cronJobUuid}/output` | Last output of a cron job |
| GET | `/servers/{uuid}/php-versions` | Installed PHP versions |
| GET | `/servers/{uuid}/php-versions/available` | Installable PHP versions |
| GET | `/servers/{uuid}/php-versions/patch-info` | Available PHP patch info |
| GET | `/servers/{uuid}/services` | Server services + status (nginx, mysql, docker, …) |
| GET | `/servers/{uuid}/supervisor-processes` | Supervisor-managed processes (paginated) |
| GET | `/servers/{uuid}/firewall-rules` | Firewall rules |
| GET | `/servers/{uuid}/firewall/ssh-restriction-status` | Is SSH IP-restricted? |
| GET | `/servers/{uuid}/fail2ban/banned-ips` | Currently banned IPs |
| GET | `/servers/{uuid}/snapshots` | Site snapshots stored on this server (paginated) |
| GET | `/servers/{uuid}/sudo-users` | Sudo users (paginated) |

### Servers — write (23)

| Method | Path | Notes |
|--------|------|-------|
| POST | `/servers/{uuid}/reboot` | Async (202) — affects ALL sites |
| POST | `/servers/{uuid}/sites/wordpress` | **WordPress only** — see request body below |
| POST | `/servers/{uuid}/cron-jobs` | Create cron — `user, frequency, pattern, command` |
| PUT | `/servers/{uuid}/cron-jobs/{cronJobUuid}` | Update cron |
| DELETE | `/servers/{uuid}/cron-jobs/{cronJobUuid}` | Delete cron |
| POST | `/servers/{uuid}/cron-jobs/{cronJobUuid}/execute` | Run cron now |
| POST | `/servers/{uuid}/php-versions` | Install PHP — `php_version` |
| DELETE | `/servers/{uuid}/php-versions` | Uninstall PHP — `php_version` (in body) |
| POST | `/servers/{uuid}/php-versions/{version}/default` | Set default PHP |
| POST | `/servers/{uuid}/php-versions/{version}/opcache` | Toggle OPcache — `enabled` |
| POST | `/servers/{uuid}/php-versions/{version}/patch` | Apply PHP patch |
| POST | `/servers/{uuid}/services/restart` | Restart service — `service, version?` |
| POST | `/servers/{uuid}/services/disable` | Disable service — `service, version?` |
| POST | `/servers/{uuid}/firewall-rules` | Create rule — `name, protocol, traffic, port?, ip_address?` |
| DELETE | `/servers/{uuid}/firewall-rules/{firewallRuleUuid}` | Delete rule |
| POST | `/servers/{uuid}/firewall-rules/{firewallRuleUuid}/enable` | Enable rule |
| POST | `/servers/{uuid}/firewall-rules/{firewallRuleUuid}/disable` | Disable rule |
| POST | `/servers/{uuid}/firewall/whitelist-caller-ip` | Whitelist the IP making the call |
| POST | `/servers/{uuid}/firewall/whitelist-xcloud-ips` | Whitelist xCloud infra IPs |
| POST | `/servers/{uuid}/fail2ban/banned-ips` | Ban IPs — `ip_addresses[]` |
| DELETE | `/servers/{uuid}/fail2ban/banned-ips/{ip}` | Unban an IP |
| POST | `/servers/{uuid}/sudo-users` | Create/update — `username, ssh_public_keys[], password?, is_temporary?` |
| DELETE | `/servers/{uuid}/sudo-users/{sudo_user_uuid}` | Remove sudo user |

### Sites — read (30)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/sites` | All sites — `?status=&type=&server_uuid=&search=&page=&per_page=` |
| GET | `/sites/{uuid}` | Full site detail |
| GET | `/sites/{uuid}/status` | Provisioning state (NOT live HTTP — see gotchas) |
| GET | `/sites/{uuid}/monitoring` | Site stats (float time-series) |
| GET | `/sites/{uuid}/monitoring/history` | `?range=24h\|7d` (default `7d`) |
| GET | `/sites/{uuid}/events` | Activity timeline (paginated) |
| GET | `/sites/{uuid}/deployment-logs` | Last deploy outputs |
| GET | `/sites/{uuid}/access-logs` | `?type=access\|nginx\|lsws` (default access), `?limit=` |
| GET | `/sites/{uuid}/git` | Repo URL, branch, deploy script, `env_file_content` (secrets!) |
| GET | `/sites/{uuid}/ssh` | SSH/SFTP config |
| GET | `/sites/{uuid}/ssh-keys` | SSH keys attached to the site |
| GET | `/sites/{uuid}/domain` | Primary domain DNS check |
| GET | `/sites/{uuid}/domain/status` | Domain-change provisioning status |
| GET | `/sites/{uuid}/domains` | All domains: `{primary, aliases, redirects, counts}` |
| GET | `/sites/{uuid}/ssl` | Primary cert summary |
| GET | `/sites/{uuid}/ssl-certificates` | All certs for the site |
| GET | `/sites/{uuid}/backups` | Backup history (the old 500 bug is fixed) |
| GET | `/sites/{uuid}/backup-count` | Number of backups |
| GET | `/sites/{uuid}/backup-settings` | Local/remote backup schedule config |
| GET | `/sites/{uuid}/backup-status` | `{local:{…}, remote:{…}}` last-backup state |
| GET | `/sites/{uuid}/cache/settings` | Cache configuration |
| GET | `/sites/{uuid}/cron-jobs` | Site cron (paginated) |
| GET | `/sites/{uuid}/cron-jobs/{cronJobUuid}/output` | Cron output |
| GET | `/sites/{uuid}/redirections` | `{items, count}` |
| GET | `/sites/{uuid}/web-rules` | Headers + redirects: `{items, counts}` |
| GET | `/sites/{uuid}/ip-access` | IP allow/deny rules |
| GET | `/sites/{uuid}/custom-nginx` | Custom nginx config blocks |
| GET | `/sites/{uuid}/site-scripts` | Deploy/site scripts |
| GET | `/sites/{uuid}/snapshots` | Site snapshots (paginated) |
| GET | `/sites/{uuid}/staging-sites` | Staging environments `{items, count}` |

### Sites — write (13)

| Method | Path | Notes |
|--------|------|-------|
| PUT | `/sites/{uuid}/ssh` | **Replaces** SSH config — `authentication_mode, ssh_public_keys[]?, password?` |
| PUT | `/sites/{uuid}/git` | Update deploy settings — `git_branch, run_after_deployment, …` |
| POST | `/sites/{uuid}/git/deploy` | Trigger a git deployment (async) |
| POST | `/sites/{uuid}/backup` | Trigger backup — `type?` (async 202) |
| POST | `/sites/{uuid}/cache/purge` | Full-page cache flush |
| POST | `/sites/{uuid}/cache/purge-all` | Purge every cache layer |
| POST | `/sites/{uuid}/ssl-certificates` | Issue/upload cert — `provider, certificate?, private_key?, force?` |
| POST | `/sites/{uuid}/ssl/renew` | Renew cert — `force?` |
| POST | `/sites/{uuid}/cron-jobs` | Create cron — `frequency, pattern?, command` |
| PUT | `/sites/{uuid}/cron-jobs/{cronJobUuid}` | Update cron |
| DELETE | `/sites/{uuid}/cron-jobs/{cronJobUuid}` | Delete cron |
| POST | `/sites/{uuid}/cron-jobs/{cronJobUuid}/execute` | Run cron now |
| POST | `/sites/{uuid}/rescue` | Repair site — toggles: `regenerate_nginx, reinstall_php, repair_node, repair_pm2, restart_pm2, …` |

### Sites — WordPress, read-only (4)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/sites/{uuid}/wordpress/status` | WP site health |
| GET | `/sites/{uuid}/wordpress/plugins` | `?status=all\|active\|inactive`, `?update_status=`, `?search=` |
| GET | `/sites/{uuid}/wordpress/themes` | Same filters as plugins |
| GET | `/sites/{uuid}/wordpress/updates` | Core/plugin/theme update summary — `?include_security_only=` |

### WordPress actions, write (5)

| Method | Path | Notes |
|--------|------|-------|
| POST | `/sites/{uuid}/magic-login` | One-time admin login URL — `login_as?` (username) |
| POST | `/sites/{uuid}/wordpress/activate` | `type` (plugin\|theme), `slugs[]`, `backup_before_action?` |
| POST | `/sites/{uuid}/wordpress/update` | `type`, `slugs[]?` (omit = all), `backup_before_update?` |
| POST | `/sites/{uuid}/wordpress/refresh` | Re-scan installed items |
| POST | `/sites/{uuid}/wp-debug` | Toggle WP_DEBUG — `enabled` |

### Vulnerabilities (6)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/vulnerabilities` | Team-wide rollup — `?severity=&source=&include_ignored=&page=&per_page=` |
| GET | `/sites/{uuid}/vulnerabilities` | Per-site list — same filters |
| GET | `/sites/{uuid}/vulnerabilities/count` | `{critical, high, medium, low, skipped, total}` |
| POST | `/sites/{uuid}/vulnerability-scan` | Trigger a rescan (async) |
| POST | `/sites/{uuid}/vulnerabilities/{vulnerabilityUuid}/ignore` | Mark as ignored |
| DELETE | `/sites/{uuid}/vulnerabilities/{vulnerabilityUuid}/ignore` | Un-ignore |

`severity`: `all\|critical\|high\|medium\|low\|unknown` · `source`: `all\|patchstack\|wordfence`

### SSL certificates — top-level resource (3)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/ssl-certificates/{uuid}` | Cert detail |
| GET | `/ssl-certificates/{uuid}/status` | Issuance/renewal status |
| DELETE | `/ssl-certificates/{uuid}` | Delete a cert |

(Create/renew live under the site: `POST /sites/{uuid}/ssl-certificates`, `POST /sites/{uuid}/ssl/renew`.)

### PageSpeed (3)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/sites/{uuid}/pagespeed` | Latest snapshot `{mobile, desktop}` (null until first scan) |
| GET | `/sites/{uuid}/pagespeed/history` | `?strategy=mobile\|desktop`, paginated |
| POST | `/sites/{uuid}/pagespeed/scan` | Trigger a PageSpeed Insights scan (async) |

### Blueprints & integrations (2)

| Method | Path | Notes |
|--------|------|-------|
| GET | `/blueprints` | WordPress site blueprints (paginated) |
| GET | `/integrations/cloudflare` | Connected Cloudflare accounts |

---

## Key request bodies (write endpoints)

```jsonc
// POST /servers/{uuid}/sites/wordpress  — create WordPress site (the only create endpoint)
{
  "mode": "live",                 // REQUIRED: "live" | "demo"
  "title": "My Site",             // REQUIRED
  "domain": "example.com",         // omit in demo mode for an auto subdomain
  "php_version": "8.3",
  "wordpress_version": "latest",
  "ssl": { },                     // object; provider/options
  "multisite": { },
  "blueprint_uuid": "…",          // clone from a blueprint
  "snapshot_uuid": "…",           // or restore from a snapshot
  "cache": { },
  "cloudflare": false,
  "additional_domains": ["www.example.com"],
  "deploy_script": "…",
  "tags": ["client-x"]
}

// POST /sites/{uuid}/cron-jobs        { "frequency": "daily", "pattern": "0 3 * * *", "command": "wp cron event run --due-now" }
// POST /servers/{uuid}/cron-jobs      { "user": "site_user", "frequency": "daily", "pattern": "...", "command": "..." }
// POST /servers/{uuid}/firewall-rules { "name": "Allow API", "protocol": "tcp", "traffic": "allow", "port": "443", "ip_address": "1.2.3.4" }
// POST /servers/{uuid}/fail2ban/banned-ips           { "ip_addresses": ["1.2.3.4", "5.6.7.8"] }
// POST /servers/{uuid}/php-versions                  { "php_version": "8.3" }
// POST /servers/{uuid}/php-versions/8.3/opcache      { "enabled": true }
// POST /servers/{uuid}/services/restart              { "service": "nginx", "version": null }
// POST /sites/{uuid}/wordpress/update                { "type": "plugin", "slugs": ["woocommerce"], "backup_before_update": true }
// POST /sites/{uuid}/wordpress/activate              { "type": "plugin", "slugs": ["woocommerce"], "backup_before_action": true }
// POST /sites/{uuid}/wp-debug                        { "enabled": true }
// POST /sites/{uuid}/magic-login                     { "login_as": "admin" }
// POST /sites/{uuid}/ssl-certificates                { "provider": "lets_encrypt", "force": false }
// POST /sites/{uuid}/ssl/renew                       { "force": true }
// PUT  /sites/{uuid}/git    { "git_branch": "main", "run_after_deployment": true, "enable_push_deploy": true, "deploy_script": "...", "restart_services": true }
// POST /sites/{uuid}/rescue { "regenerate_nginx": true, "restart_nginx": true, "repair_node": true, "repair_pm2": true, "restart_pm2": true }
```

## Pagination

For list endpoints, default page size is small (10–15). Use `?per_page=50&page=1`. `per_page` cap is server-side; check `pagination.last_page` to iterate.

## Async operations (return 202 — poll to confirm)

Confirm via `/sites/{uuid}/events` (site work) or `/servers/{uuid}/tasks` (server work) until the matching entry shows `completed`/`finished` or `failed`.

- `POST /servers/{uuid}/reboot`
- `POST /servers/{uuid}/sites/wordpress`
- `POST /servers/{uuid}/php-versions` (install) and `/{version}/patch`
- `POST /sites/{uuid}/backup`
- `POST /sites/{uuid}/git/deploy`
- `POST /sites/{uuid}/vulnerability-scan`
- `POST /sites/{uuid}/pagespeed/scan`
- `POST /sites/{uuid}/wordpress/update` / `activate` (may run async)

## Quick examples

```bash
# Find a site by domain (no server-side domain filter — use ?search= then match locally)
xc_get "/sites?search=mydomain&per_page=100" | sed '$d' | python -c "
import sys, json
data = json.load(sys.stdin)['data']['items']
print(json.dumps([s for s in data if 'mydomain.com' in s['domain_name']], indent=2))
"

# Vulnerability posture for the whole team
xc_get "/vulnerabilities?severity=critical&per_page=100" | sed '$d' | python -m json.tool | head -40

# Trigger a deploy and watch events
xc_send POST "/sites/$SITE_UUID/git/deploy"
sleep 5
xc_get "/sites/$SITE_UUID/events" | sed '$d' | python -m json.tool | head -40
```

## Gotchas (verified against live API 2026-06-18)

- **`GET /sites/{uuid}/backups` works now** — the earlier beta 500 error is fixed.
- **`GET /servers/{uuid}/databases` was REMOVED** (returns 404). The public API no longer lists databases; use `/xcloud-db` (SSH/mysql) instead.
- **`DELETE /user/tokens/{tokenId}` → `{tokenUuid}`** — the path param was renamed.
- **`monitoring/history` `range` only accepts `24h` or `7d`** — any other value returns 422 `"The selected range is invalid."`.
- **WordPress endpoints don't hard-fail on non-WP sites** — they return empty/null `data` (e.g. `wordpress/updates` on a Node.js site returns null versions). Check `site.type == "wordpress"` before treating WP data as meaningful.
- **Site `type` is opaque** — live values include `wordpress`, `nodejs`, `custom-php`, `phpmyadmin`, `oneclick`, `static`. Never error on an unknown type.
- **`/sites/{uuid}/status` is provisioning state, not live HTTP** — do an independent `curl -sI` to the public domain for true uptime.
- **`PUT /sites/{uuid}/ssh` REPLACES the full SSH key list** — never call without explicit user confirmation; locking yourself out is real.
- **Server monitoring values are strings** (`"57%"`), site monitoring values are floats.
- **Only WordPress site creation is exposed** — there is no public endpoint to create a server or a non-WordPress site. (The marketing changelog mentions "auto-provision servers"; the v1.0.0 spec does not yet expose it.)
- SSH port is not exposed by the API; xCloud convention is port 22.

## Failure handling pattern

Every command should:

1. Load token via the helper at the top.
2. Call API. Capture full response + HTTP code.
3. If HTTP ≥ 400 → show `message` (and `errors`) from the body, suggest a fix (per status table), abort.
4. On 401/403 → point at `token-setup.md` (403 = missing scope).
5. On 202 → tell the user the op is queued and (optionally) poll events/tasks.
