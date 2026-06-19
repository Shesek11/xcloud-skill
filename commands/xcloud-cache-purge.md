---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Purge the full-page cache for an xCloud site. Use after content changes that aren't reflecting (CSS, posts, plugin updates). Effective on Nginx-cached sites.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## Purge Full-Page Cache

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. If absent, run `/xcloud-init`.

### Step 2: Pick scope + confirm

Use AskUserQuestion (skip only if the user already specified):
- **Full-page cache** (default) — flush the page cache (`/cache/purge`)
- **All caches** — flush every cache layer xCloud manages (`/cache/purge-all`)

You can also inspect what's configured first: `GET /sites/{uuid}/cache/settings`.

Then confirm:

```
Purge [full-page | ALL] cache for [domain_name]?
This will force a fresh render on the next request — minor traffic spike possible.
Proceed? (yes/no)
```

### Step 3: Send purge

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
B="https://app.xcloud.host/api/v1"
# Full-page:
curl -sS -w "\n__HTTP__:%{http_code}" -X POST \
  -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" \
  "$B/sites/$SITE_UUID/cache/purge"
# All caches:
curl -sS -w "\n__HTTP__:%{http_code}" -X POST \
  -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" \
  "$B/sites/$SITE_UUID/cache/purge-all"
# Inspect cache config:
curl -sS -H "Authorization: Bearer $XCLOUD_TOKEN" -H "Accept: application/json" \
  "$B/sites/$SITE_UUID/cache/settings"
```

Expected: HTTP **200** or **202**.

### Step 4: Result handling

- HTTP 200/202 → Show "Cache purged for [domain_name]."
- HTTP 403 → Token lacks `write:sites` scope.
- HTTP 422 → Site type does not support cache (e.g. static, Node.js without server-side cache). Surface the API message verbatim.
- HTTP 404 → Wrong UUID or site no longer accessible.

### Step 5: Optional — verify

Offer to fetch the homepage and check the response headers for cache hits:

```bash
curl -sI "https://[domain_name]" 2>/dev/null | grep -iE 'x-cache|cf-cache-status|age'
```

After a purge, expect `MISS` on the first hit, then `HIT` on subsequent hits.

## Important Notes

- This only purges the **xCloud full-page cache** at the edge / web server level.
- Cloudflare cache is separate — purge via Cloudflare dashboard or CLI if you have it in front.
- Browser caches are not purged. Hard-refresh (Ctrl+Shift+R) on the client side if needed.
- For Node.js sites, full-page cache may not be configured — the API will return a clear message in that case.
