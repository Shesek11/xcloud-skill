# Changelog

All notable changes to the xCloud plugin are documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [2.1.0] - 2026-06-19

Consolidated the xCloud skill into a single canonical home: **this repo**
(`xcloud-skill`), restructured as a proper plugin and cloned to
`~/.claude/plugins/xcloud/`. Folded in all accumulated xCloud knowledge.

### Added
- `KNOWLEDGE.md` — accumulated gotchas, the marketing-vs-reality API check, parameter enums, account snapshot, and a Node.js persistence summary.
- `skills/xcloud-nodejs-persistence/` — PM2 reboot-persistence playbook, bundled into the repo.
- `commands/xcloud-new-site.md` — guided WordPress / Node.js site creation (SSH), retained from the original skill.

### Changed
- Restructured into a clean plugin layout: `commands/`, `references/`, `skills/`, `.claude-plugin/plugin.json` (previously a flat command dump mixed with unrelated skills).
- This repo is now the single source of truth, cloned to `~/.claude/plugins/xcloud/`; the duplicate copy formerly maintained under `claude-config` has been retired.

### Removed
- Non-xcloud commands (`gsd/`, `audit`, `critique`, `cybersec`, `extract`, `init`, `resume-session`, `security-review`, `status`) that were incidentally in this repo — they live elsewhere now.

## [2.0.0] - 2026-06-18

Tracks the xCloud Public API graduating from beta to **v1.0.0**, which grew from
33 to **111 operations**. Refreshed the spec snapshot, reference docs, and added
commands for every major new capability area.

### Added
- **`/xcloud-security`** — vulnerability scanning (team rollup + per-site list/count, trigger rescan, ignore/un-ignore), firewall rules (create/enable/disable/delete, whitelist caller/xCloud IPs), and fail2ban (list/ban/unban IPs).
- **`/xcloud-wp`** — WordPress management: list plugins/themes/updates, site health, activate, update, refresh, toggle WP_DEBUG, and generate a one-time magic-login URL.
- **`/xcloud-cron`** — cron jobs for sites and servers: list, create, update, delete, run-now, and view output.
- **`/xcloud-php`** — PHP version management: list installed/available, install, uninstall, set default, toggle OPcache, apply patches.
- **`/xcloud-deploy`** — trigger a git deployment via the API and view/update deploy settings (branch, push-deploy, deploy script).
- **`/xcloud-services`** — list server services, restart/disable a service, and list supervisor-managed processes.
- **`/xcloud-pagespeed`** — latest PageSpeed Insights snapshot, history, and trigger a scan.
- **`/xcloud-snapshots`** — list snapshots for a site or across a server.
- **`/xcloud-staging`** — list staging environments for a site.
- **`/xcloud-redirects`** — redirections, web-rules (custom headers/redirects), and IP access rules.
- `CHANGELOG.md` and a `version` field in `plugin.json` (`2.0.0`).

### Changed
- **`references/openapi.yaml`** replaced with the live v1.0.0 spec (111 operations, out of beta).
- **`references/api-reference.md`** rewritten: full 111-operation catalog across 11 categories, request-body cheatsheet, parameter enums, and refreshed gotchas.
- **`/xcloud-ssl`** enhanced from a read-only summary into the full SSL certificate resource: list, detail, status, **create, renew, delete**.
- **`/xcloud-backup`** enhanced with backup settings, status, and count alongside list/trigger.
- **`/xcloud-cache-purge`** now also offers purge-all (every cache layer), not just full-page.
- **`/xcloud`** hub updated to list and route to all new commands.
- `README.md` updated: command tables, capabilities, files list, and beta → v1.0.0 status.

### Fixed
- `GET /sites/{uuid}/backups` no longer 500s (the beta-era bug is resolved upstream); `/xcloud-backup` reads it directly again.

### Removed
- References to `GET /servers/{uuid}/databases` — the endpoint was removed from the API (404). Use `/xcloud-db` (SSH/mysql) for database access.

### Notes
- The OpenAPI v1.0.0 spec declares only a `bearerAuth` scheme and no longer enumerates per-endpoint scopes; token scopes still apply at creation time. A `*` token is simplest for full plugin coverage.
- Despite changelog marketing about "auto-provisioning servers", the v1.0.0 spec still exposes **only WordPress site creation** — no server-create or non-WordPress site-create endpoint.

## [1.0.0] - 2026-04-26

### Added
- Initial API-driven overhaul: token setup, auto-discovery init/migration, and commands for list, info, events, deploy-log, monitoring, backup, cache-purge, ssl, reboot, plus enriched status/push and SSH/git/MariaDB workflow.
