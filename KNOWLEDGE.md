# xCloud Knowledge Base

Accumulated, hard-won knowledge about the xCloud Public API and hosting platform.
This is the durable "what we actually learned" companion to `references/` (the formal
spec) and `commands/` (the slash commands). Last verified **2026-06-19**.

---

## API at a glance

- **xCloud Public API v1.0.0** — out of beta as of June 2026, **free for all users**.
- Base URL: `https://app.xcloud.host/api/v1`
- **111 operations** across 11 categories (Servers, Sites, Sites-WordPress, WordPress Actions, Vulnerabilities, SSL Certificates, PageSpeed, Blueprints, Integrations, User, Health).
- Interactive docs (Scalar) live at `https://app.xcloud.host/api/v1/docs`; the spec is embedded as YAML in that page (no separate downloadable `.json`/`.yaml` route — `references/openapi.yaml` is a captured snapshot).

## Auth & scopes

- Sanctum bearer token, format `34|...` (51 chars). Stored at `~/.xcloud/token` (chmod 600, **no trailing newline** — use `printf`, not `echo`) or `$XCLOUD_API_TOKEN`.
- Create at **Account → API Tokens** (`/user/api-tokens`). Scopes: `read:servers`, `write:servers`, `read:sites`, `write:sites`, or `*`.
- The v1.0.0 OpenAPI declares only a `bearerAuth` scheme and **no longer lists scopes per endpoint**, but scopes are still enforced. **403 = missing scope** → recreate the token (use `*` if unsure).

## Reality check — marketing vs. the actual API

The June 2026 launch email overstates the API. Verified against the live spec:

| Marketing claim | Reality |
|---|---|
| "Provision servers programmatically, no dashboard" | ❌ **Not in the API.** No `POST /servers`. Every server endpoint needs an existing server `{uuid}`. Server creation is dashboard-only. |
| "Deploy fully configured sites in a single API call" | ⚠️ **WordPress only** — `POST /servers/{uuid}/sites/wordpress` (rich body). No Node.js/PHP/static site creation. |
| "Add/remove SSH and sudo access on demand" | ✅ `PUT /sites/{uuid}/ssh`, `POST/DELETE /servers/{uuid}/sudo-users`. |
| "Pull live server and site data into any system" | ✅ Extensive read endpoints. |
| "Spin up environments for every client" | ⚠️ Only as WordPress sites; staging is **read-only** via the API. |
| "AI Agent Skills / MCP on the way" | Future ("in the pipeline") — not present. (This repo already provides the agent skills.) |

**Bottom line:** ~80% accurate. The flagship "provision servers" claim is not real today.

## Gotchas (verified live)

- `GET /sites/{uuid}/backups` — the old beta **500 bug is fixed** (returns 200).
- `GET /servers/{uuid}/databases` — **removed** (404). Use SSH/mysql (`/xcloud-db`).
- `DELETE /user/tokens/{tokenId}` → renamed **`{tokenUuid}`**.
- `monitoring/history` `range` accepts only **`24h` or `7d`** (default `7d`); anything else → 422.
- WordPress endpoints **don't hard-fail on non-WP sites** — they return empty/null. Check `site.type == "wordpress"` first.
- Site `type` is **opaque** — seen: `wordpress`, `nodejs`, `custom-php`, `phpmyadmin`, `oneclick`, `static`. Never error on an unknown type.
- `GET /sites/{uuid}/status` = **provisioning state, not live HTTP**. Do an independent `curl -sI` for true uptime.
- `PUT /sites/{uuid}/ssh` **replaces the entire SSH key list** — never call without explicit confirmation (lockout risk).
- Server monitoring values are **strings** (`"57%"`); site monitoring values are **floats** (time-series).
- Async ops return **202** (reboot, backup, git deploy, php install/patch, vuln scan, pagespeed scan, WP update/activate) — confirm via `/sites/{uuid}/events` or `/servers/{uuid}/tasks`.

## Parameter enums (quick reference)

- `monitoring/history` `range`: `24h | 7d`
- vulnerabilities `severity`: `all | critical | high | medium | low | unknown`; `source`: `all | patchstack | wordfence`
- `wordpress/plugins`/`themes` `status`: `all | active | inactive`
- `access-logs` `type`: `access | nginx | lsws`
- `pagespeed/history` `strategy`: `mobile | desktop`
- WordPress site create `mode`: `live | demo`

## What the API can NOT do (still needs SSH/panel)

- Create a server (no provisioning endpoint).
- Create a non-WordPress site (only WordPress creation is exposed).
- Run arbitrary shell commands; direct DB queries; direct file edits → use `/xcloud-ssh`, `/xcloud-db`.
- Restore a backup / create-or-restore a snapshot / create staging → dashboard only (the API lists these but doesn't mutate them).

## Node.js persistence (the #1 recurring xCloud pain)

Node.js apps 502 after server reboot because xCloud has no built-in Node auto-start and `pm2 startup` needs sudo (unavailable on shared hosting). Fix = three independent mechanisms:

1. **Deploy script** (git push): `... && pm2 start ecosystem.config.cjs && pm2 save` (use `pm2 start`, not `pm2 restart`).
2. **`@reboot` cron** (server restart): cron line that `cd`s to the app and runs `pm2 start ... && pm2 save`.
3. **`autorestart: true`** in the ecosystem config (process crash).

Full playbook (incl. the dual-ecosystem `EADDRINUSE` port-conflict variant): **`skills/xcloud-nodejs-persistence/SKILL.md`**.

## Account snapshot (2026-06)

- **2 servers**: `Apps` (Vultr, Tel Aviv, 64.177.67.166) and `Docker` (64.176.163.72).
- **~17 sites**, mostly **Node.js**; WordPress is rare for this account.
- `Apps` runs hot (small instance, RAM/swap pressure) — watch `/xcloud-monitoring`.

## This repo is the home

`git@github.com:Shesek11/xcloud-skill.git` is the single canonical home for the xCloud
skill. It is cloned to `~/.claude/plugins/xcloud/` and loads as the live plugin. Edit
here, push here. (Historically the work was split between this repo and the `claude-config`
plugin copy — that duplication has been retired.)
