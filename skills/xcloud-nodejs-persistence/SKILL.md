---
name: xcloud-nodejs-persistence
description: |
  This skill should be used when a Node.js app on xCloud returns 502 Bad Gateway
  after server restart, PM2 process list is empty after reboot, pm2 restart all
  does nothing, pm2 startup fails without sudo, deploy script runs but app does
  not start, or PM2 shows EADDRINUSE port conflict from dual ecosystem configs.
  Also triggered by "my xCloud site is down after reboot", "xCloud Node app
  not starting", or "PM2 processes disappear on xCloud restart".
  Covers PM2 persistence via @reboot cron, ecosystem.config.js with dotenv,
  and deploy script best practices for xCloud hosting.
version: 1.2.0
---

# xCloud Node.js PM2 Persistence

## Problem

Node.js apps on xCloud go down (502 Bad Gateway) after every server restart because:
1. xCloud has no built-in mechanism to auto-start Node.js apps on server reboot
2. `pm2 startup` requires `sudo` which is unavailable on shared hosting
3. The default deploy script template uses `pm2 restart all` which does nothing when PM2 daemon starts fresh with no saved processes

## Diagnosis

Before applying this fix, confirm the issue by SSHing to the server and running:
```bash
pm2 list                    # Empty table = persistence problem
pm2 startup                 # "sudo: a password is required" = confirms no startup support
crontab -l                  # No @reboot entry = no boot persistence
```

## Root Cause

xCloud manages Node.js apps via PM2 with two separate files:

| File | Location | Used By |
|------|----------|---------|
| **xCloud's config** | `~/[site-name]/ecosystem.config.js` | xCloud's own process management |
| **App's config** | `/var/www/[site-name]/ecosystem.config.cjs` | The deploy script & PM2 |

**Critical insight**: The deploy script (xCloud Git settings) runs only on **git push**, NOT on server restart. These are two completely separate events that need separate solutions.

## Solution

### 1. Fix the Deploy Script (for git push)

In xCloud panel → Sites → Git → Deploy Script, use `pm2 start` (NOT `pm2 restart`):

```bash
cd /var/www/[site-name] && npm install --production && npm run build && pm2 start ecosystem.config.cjs && pm2 save
```

**Why `pm2 start` not `pm2 restart`**: `pm2 start` is idempotent - if the process exists it restarts it, if not it creates it. `pm2 restart all` fails when PM2 has no processes.

**Enable the toggle**: "Run this script after every site deployment" must be **ON**.

> **Note**: If xCloud already manages the PM2 process via its own `~/[site-name]/ecosystem.config.js`, use the Variant approach at the bottom of this document instead — the two strategies are mutually exclusive.

### 2. Add @reboot Cron (for server restart)

Since `pm2 startup` needs sudo (unavailable on shared hosting), use cron:

```bash
ssh user@host "(crontab -l 2>/dev/null; echo '@reboot PATH=/usr/local/bin:/usr/bin:/bin; cd /var/www/[site-name] && /usr/bin/pm2 start ecosystem.config.cjs --env production && /usr/bin/pm2 save') | crontab -"
```

Verify:
```bash
ssh user@host "crontab -l"
```

### 3. Update xCloud's ecosystem.config.js (for dotenv support)

xCloud's default config at `~/[site-name]/ecosystem.config.js` uses `npm start` which does NOT preload environment variables. Update it:

```javascript
module.exports = {
    apps: [{
        name: "nodejs-[site-name]",
        script: "./server/server.js",  // direct script, not npm start
        cwd: "/var/www/[site-name]",
        node_args: "-r dotenv/config",  // CRITICAL: preload .env
        // Note: PM2 does not natively support env_file. Environment loading
        // happens via node_args: "-r dotenv/config" above.
        env: {
            NODE_ENV: "production",
            PORT: 3000,  // Must match xCloud's Nginx proxy port for this site
            DOTENV_CONFIG_PATH: ".env"
        },
        watch: false,
        autorestart: true,
        max_memory_restart: "500M",
        max_restarts: 10,
        min_uptime: "10s"
    }]
};
```

### 4. Always `pm2 save` After Starting

After any `pm2 start`, always run `pm2 save` to persist the process list to `~/.pm2/dump.pm2`. This dump file is what `pm2 resurrect` reads, providing an additional recovery path.

## Three Scenarios Covered

| Scenario | Mechanism |
|----------|-----------|
| **Git push to main** | Deploy script: install, build, pm2 start, pm2 save |
| **Server restart** | @reboot cron job starts PM2 with ecosystem config |
| **PM2 process crash** | `autorestart: true` in ecosystem config |

## Verification

After setting up all three mechanisms:

1. **Test deploy**: Push a commit → verify site comes up
2. **Test server restart**: Restart server from xCloud panel → wait ~20-30 seconds → verify site comes up
3. **Check**: `ssh user@host "uptime && pm2 list"` - should show app online shortly after boot

## Notes

- The @reboot cron takes ~15-30 seconds to bring up the app after server boot
- `pm2 start` with an ecosystem config file is always safe to run (idempotent)
- If app uses ES modules, environment variables from `.env` are NOT available at import time unless using `-r dotenv/config` node_args or lazy initialization patterns
- xCloud's web root for Node.js apps is `/var/www/[site-name]/`
- xCloud's PM2 config location is `~/[site-name]/ecosystem.config.js`

## Variant: Dual Ecosystem Config Port Conflict (EADDRINUSE)

**Symptom**: `pm2 list` shows process in `errored` state with restart count climbing.
PM2 error log shows: `Error: listen EADDRINUSE: address already in use :::3000`

**Root Cause**: Two PM2 processes trying to bind the same port:
1. xCloud's own `~/[site-name]/ecosystem.config.js` starts `nodejs-[site-name]` on port X
2. Deploy script runs `pm2 start ecosystem.config.cjs` which starts a second process `[app-name]` also on port X

**Diagnosis**:
```bash
pm2 list                          # Shows two process names, one errored
ss -tlnp | grep :[PORT]           # Port already in use
cat ~/[site-name]/ecosystem.config.js  # xCloud's config (authoritative)
cat /var/www/[site-name]/ecosystem.config.cjs  # App's config (duplicate)
```

**Fix**:
1. Delete the app's `ecosystem.config.cjs` (both from server AND git repo)
2. Update xCloud's `~/[site-name]/ecosystem.config.js` to use correct PORT and load env:

```javascript
module.exports = {
    apps: [{
        name: "nodejs-[site-name]",
        script: "node_modules/.bin/next",
        args: "start",
        cwd: "/var/www/[site-name]",
        node_args: "-r dotenv/config",
        env: {
            NODE_ENV: "production",
            PORT: "2998",  // ← Must match Nginx proxy port, NOT 3000
            DOTENV_CONFIG_PATH: "/var/www/[site-name]/.env"
        },
        watch: false,
        autorestart: true,
        max_memory_restart: "500M"
    }]
};
```

3. Change deploy script to use `pm2 reload` not `pm2 start ecosystem.config.cjs`:
```bash
cd /var/www/[site-name] && npm install --production && npm run build && pm2 reload nodejs-[site-name] && pm2 save
```

4. After env changes, always reload with `--update-env`:
```bash
pm2 reload nodejs-[site-name] --update-env
```

**Key Insight**: xCloud's `~/[site-name]/ecosystem.config.js` is the **authoritative** PM2 config.
The app should NOT have its own ecosystem config — use `pm2 reload [process-name]` in deploy scripts.
The Nginx proxy port (e.g., 2998) must be set explicitly — `env_file` alone doesn't guarantee PORT loads.

## See Also

- Slash commands: `/xcloud-ssh`, `/xcloud-new-site`
- Related skill: `nextauth-v5-reverse-proxy` (for NextAuth v5 reverse proxy issues)
- ES module dotenv issue: imports hoist before `dotenv.config()` runs, use `node_args: '-r dotenv/config'` or lazy initialization
