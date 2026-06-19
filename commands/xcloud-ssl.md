---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(openssl:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Manage SSL certificates for an xCloud site — view the active cert (issuer, expiry, days left), list all certs, check issuance status, create/upload a cert, renew, or delete. Use to confirm a renewal, fix a failed cert, or spot an upcoming expiry.
---

## Context

- Site UUID: !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`

## xCloud SSL Certificates

See `plugins/xcloud/references/api-reference.md` for the auth helper.

### Step 1: Resolve site UUID

Read `.xcloud/context.md`. Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:
- **Summary** (default) — active cert: provider, status, expiry, days left
- **List certs** — all certificates attached to the site
- **Cert status** — issuance/renewal status for a specific cert
- **Create / upload** — issue a Let's Encrypt cert or upload a custom one
- **Renew** — force/renew the current cert
- **Delete** — remove a certificate

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"

# Summary (primary cert) — fields: provider, status, expires_at, hostnames
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/ssl"
# List all certs for the site
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/ssl-certificates"
# Detail / status for a specific cert (uuid from the list)
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/ssl-certificates/$CERT_UUID"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/ssl-certificates/$CERT_UUID/status"

# Create: provider e.g. "lets_encrypt" (auto) or "custom" (+ certificate & private_key PEM)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"provider":"lets_encrypt","force":false}' "$B/sites/$SITE_UUID/ssl-certificates"
# Renew
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" -d '{"force":true}' "$B/sites/$SITE_UUID/ssl/renew"
# Delete a cert
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/ssl-certificates/$CERT_UUID"
```

### Step 4: Render summary

```
─── SSL: [domain_name] ───
Provider:  [provider]              (xcloud / lets_encrypt / cloudflare / custom)
Status:    [status]                (obtained / pending / failed)
Expires:   [expires_at]            ([N] days from now)
Hostnames: [comma-separated or "n/a"]
```

For **List**, table: `# | hostnames | provider | status | expires | uuid`.

### Step 5: Severity flags

- `expires_at` < 7 days → **CRITICAL**: confirm a renewal is happening (check `/xcloud-events` for cert events); if not, renew now.
- 7–14 days → **WARNING**: monitor; auto-renew usually fires ~30 days out.
- `status: failed` → **CRITICAL**: issuance failed — almost always a DNS or HTTP-01 challenge problem. Check the cert `status` endpoint and `/xcloud-events` for the error.
- `status: pending` → in progress; should resolve in minutes.
- `provider: cloudflare` (proxied) → TLS terminates at Cloudflare; the xCloud cert may differ from what users see.

### Step 6: Confirm writes & cross-verify

- Confirm before **Create / Renew / Delete**. For a **custom** cert, the PEM `certificate` and `private_key` are secrets — never echo them back or write them to `context.md`.
- **Deleting** the active cert can break HTTPS — confirm there's a replacement.
- Optionally compare against the live cert:
  ```bash
  echo | openssl s_client -connect [domain]:443 -servername [domain] 2>/dev/null \
    | openssl x509 -noout -dates -issuer -subject 2>/dev/null
  ```
  If the API cert and the served cert disagree, DNS likely points elsewhere (Cloudflare proxy, wrong A record, old CDN).

## Important Notes

- Create/renew live under the **site** (`/sites/{uuid}/...`); get/status/delete live under the **cert** (`/ssl-certificates/{uuid}`).
- The summary payload (`/sites/{uuid}/ssl`) has no `auto_renew` field — xCloud renews internally; verify via `/xcloud-events`.
- The cert the API reports and the cert end users see diverge most often when Cloudflare is in front — use the OpenSSL check to be sure.
