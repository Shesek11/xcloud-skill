---
allowed-tools: Bash(curl:*), Bash(test:*), Bash(cat:*), Bash(tr:*), Bash(grep:*), Bash(python:*), Bash(C\:\\Users\\shimo\\AppData\\Local\\Python\\bin\\python.exe:*), Read, Glob, AskUserQuestion
description: Security operations for an xCloud site/server — scan for vulnerabilities (Patchstack/Wordfence), manage firewall rules, and ban/unban IPs via fail2ban. Use to audit security posture or harden a server.
---

## Context

- Site UUID:   !`test -f .xcloud/context.md && grep -E 'Site UUID' .xcloud/context.md | head -1`
- Server UUID: !`test -f .xcloud/context.md && grep -E 'Server UUID' .xcloud/context.md | head -1`

## xCloud Security

See `plugins/xcloud/references/api-reference.md` for the auth helper and full endpoint list.

### Step 1: Resolve UUIDs

Read `.xcloud/context.md` (Site UUID for vulnerabilities, Server UUID for firewall/fail2ban). Run `/xcloud-init` if missing.

### Step 2: Pick action

Use AskUserQuestion:

**Vulnerabilities (site-level, read):**
- **Posture** (default) — count by severity + top findings for this site
- **Team rollup** — vulnerabilities across all sites in the team
- **Rescan** — trigger a fresh scan (async)
- **Ignore / un-ignore** — suppress or restore a specific finding

**Firewall (server-level):**
- **List rules**
- **Create rule** / **Enable** / **Disable** / **Delete rule**
- **Whitelist** — caller IP or xCloud infrastructure IPs

**fail2ban (server-level):**
- **List banned IPs**
- **Ban IPs** / **Unban an IP**
- **SSH restriction status**

### Step 3: Run

```bash
XCLOUD_TOKEN="${XCLOUD_API_TOKEN:-$(tr -d '\r\n' < ~/.xcloud/token)}"
H_AUTH="Authorization: Bearer $XCLOUD_TOKEN"; H_ACC="Accept: application/json"; H_CT="Content-Type: application/json"
B="https://app.xcloud.host/api/v1"
```

**Vulnerabilities**
```bash
# Count by severity → {critical,high,medium,low,skipped,total}
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/vulnerabilities/count"
# List (filters: severity=all|critical|high|medium|low|unknown, source=all|patchstack|wordfence)
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/vulnerabilities?severity=all&per_page=50"
# Team-wide rollup
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/vulnerabilities?severity=critical&per_page=100"
# Trigger rescan (async 202)
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/vulnerability-scan"
# Ignore / un-ignore a finding (needs the vulnerability UUID from the list)
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/vulnerabilities/$VULN_UUID/ignore"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/sites/$SITE_UUID/vulnerabilities/$VULN_UUID/ignore"
```

**Firewall**
```bash
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall-rules"
# Create: protocol tcp|udp|any, traffic allow|deny
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"name":"Allow HTTPS","protocol":"tcp","traffic":"allow","port":"443","ip_address":"1.2.3.4"}' \
  "$B/servers/$SERVER_UUID/firewall-rules"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall-rules/$RULE_UUID/enable"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall-rules/$RULE_UUID/disable"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall-rules/$RULE_UUID"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall/whitelist-caller-ip"
curl -sS -X POST   -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall/whitelist-xcloud-ips"
```

**fail2ban**
```bash
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/fail2ban/banned-ips"
curl -sS -X POST -H "$H_AUTH" -H "$H_ACC" -H "$H_CT" \
  -d '{"ip_addresses":["1.2.3.4"]}' "$B/servers/$SERVER_UUID/fail2ban/banned-ips"
curl -sS -X DELETE -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/fail2ban/banned-ips/1.2.3.4"
curl -sS -H "$H_AUTH" -H "$H_ACC" "$B/servers/$SERVER_UUID/firewall/ssh-restriction-status"
```

### Step 4: Render & flag

- **Vulnerability posture** — show `critical/high/medium/low/total`. Any **critical** or **high** → flag prominently and list affected component + fixed version. `source` tells you Patchstack vs Wordfence.
- WordPress vulnerabilities only populate for WordPress sites. On non-WP sites the counts are 0 — say so rather than implying the site is clean.
- After a **rescan**, results are async — tell the user to re-run "Posture" in a minute.
- **Firewall list** — show name, protocol, port, ip, enabled/disabled. The default `SSH` rule on port 22 is expected.

### Step 5: Confirm destructive actions

ALWAYS confirm before: deleting a firewall rule, banning an IP, or whitelisting. Show exactly what will change. Note these affect the **whole server** (all sites on it).

## Important Notes

- Vulnerabilities are site-scoped; firewall + fail2ban are server-scoped (shared across every site on the server).
- Banning your own caller IP or deleting the SSH allow-rule can lock you out — double-check before applying.
- Whitelisting xCloud infrastructure IPs is the fix when the panel/agent can't reach a hardened server.
