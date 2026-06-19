---
allowed-tools: Bash(ssh:*), Bash(git:*), Bash(mkdir:*), Bash(ls:*), Bash(cat:*), Bash(npm:*), Bash(node:*), Write, Read, Glob, Grep, AskUserQuestion
description: Create a new site on an existing xCloud server - WordPress or Node.js with full guided setup
---

## Your Task: Create a New Site on xCloud

You are guiding the user through creating a new website on their existing xCloud server.
This is a 3-phase process: Preparation (automated) → Panel Setup (guided) → Post-Setup (automated).

**Important:** xCloud has NO public API. Site creation MUST go through their web panel.
Your job is to prepare everything before, guide during, and automate after.

---

## Phase 1: Gather Information

### Step 1: What Type of Site?

Use AskUserQuestion to ask:

**"What type of site do you want to create?"**
- **Node.js (SSR)** - Next.js, Nuxt.js, Express with server-side rendering
- **Node.js (CSR)** - React, Vue, static frontend served by Node
- **WordPress** - Fresh WordPress installation or clone from Git

### Step 2: Project Details

Ask the user for:

1. **Project Name** - Short identifier (e.g., "my-app")
2. **GitHub Repository** - Existing repo URL, or should we create one?
3. **Git Branch** - Main branch name (default: "main")
4. **Domain Strategy** - Start with staging/demo domain, or use a live domain?

### Step 3: Existing Server Selection

Ask: **"Which server should this site be on?"**

Check if user has existing `.xcloud/context.md` files in any projects:
```bash
find /c/dev -name "context.md" -path "*/.xcloud/*" 2>/dev/null | head -10
```

If found, extract server details from existing context files to suggest.
Otherwise, ask for:
- **SSH Host** and **SSH Port**
- **SSH Username**
- The server must already be provisioned in xCloud with the appropriate stack (NGINX or OLS)

### Step 4: Site-Type-Specific Information

**For Node.js (SSR):**
- **Port** - What port does the app listen on? (default: 3000)
- **Start Command** - How to start the app? (e.g., `npm start`, `node server.js`, `yarn start`)
- **Web Root Path** - Subdirectory for app files? (blank = root)
- **Node.js Version** - Which version? (18, 20, 22)
- **Build Command** - Build step for deployment? (e.g., `npm run build`)
- **Environment Variables** - Any .env content needed?

**For Node.js (CSR):**
- Same as SSR but toggle "Server-Side Rendering App" OFF in panel
- **Build Output Directory** - Where does the build output go? (e.g., `dist`, `build`)

**For WordPress:**
- **PHP Version** - 7.4, 8.0, 8.1, 8.2, 8.3?
- **Multisite?** - Yes/No
- **Enable Caching?** - Full Page Cache + Redis Object Cache (recommended: yes)
- **Blueprint** - Any pre-saved blueprint to use?

### Step 5: Database

Ask: **"Do you need a database?"**
- **Create New Database** (recommended) - xCloud creates it automatically
- **Use Existing Database** - Provide host, port, user, name, password
- **No Database** - Skip (Node.js CSR only)

If creating new, suggest auto-generated names based on project name:
- DB Name: `[project]_db`
- DB User: `[project]_user`
- Let xCloud auto-generate the password

---

## Phase 2: Pre-Panel Preparation (Automated)

### Step A: Verify/Prepare Git Repository

**If repo exists:**
```bash
git clone <repo-url> <local-path>
```
Verify the repo has the expected structure.

**If creating new repo:**
For Node.js - create minimal structure:
```
package.json
server.js (or appropriate entry point)
.gitignore
.env.example
README.md
```

For WordPress with Git:
```
wp-config-sample.php (REQUIRED by xCloud for Git clone)
wp-content/
.gitignore
```

**Important for WordPress Git Clone:** The repository MUST contain `wp-config-sample.php`. xCloud generates `wp-config.php` from it with database info.

### Step B: Prepare SSH Key (if needed)

Check if SSH key auth works with the target server:
```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes -p [port] [user]@[host] "echo SSH_KEY_OK" 2>/dev/null
```

If not working, guide through SSH key setup (refer to xcloud-init.md SSH Key Setup Guide).

### Step C: Prepare Deployment Script

Generate a deployment script based on site type:

**For Node.js:**
```bash
cd /home/[user]/[site-path]
npm install --production
npm run build  # if applicable
pm2 restart ecosystem.config.js  # or appropriate restart
```

**For WordPress:**
Usually not needed. xCloud handles WP setup automatically.

### Step D: Prepare .env Content

For Node.js, prepare the .env file content:
```
NODE_ENV=production
PORT=[port]
DATABASE_URL=mysql://[db_user]:[db_pass]@localhost:[db_port]/[db_name]
# Add other environment variables as needed
```

Tell user: **"Save this - you'll paste it in the xCloud panel."**

### Step E: Generate Panel Cheat Sheet

Create a summary of everything the user needs to enter in the panel:

```
========================================
 xCloud Panel Cheat Sheet
========================================

SITE TYPE: [Node.js SSR / Node.js CSR / WordPress]

1. Add New Site → Select Server: [server-name]
2. Application Tab: [Node.js / WordPress]

--- FOR NODE.JS ---
3. Node.js Version: [version]
4. Server-Side Rendering: [ON/OFF]
5. Port: [port]
6. Start Command: [command]
7. Web Root Path: [path or blank]
8. Database: Create New Database
   - DB Name: [name]
   - DB User: [user]
   - Password: [auto-generated, save it!]

--- FOR WORDPRESS ---
3. Install New WordPress Website
4. Site Title: [title]
5. Domain: [staging or live domain]
6. PHP Version: [version]
7. Caching: Full Page + Redis [ON/ON]
8. Database: Create New Database

--- GIT REPO (BOTH) ---
9. Repository Type: [Private SSH / Public HTTPS / Connected Provider]
10. Repository URL: [url]
11. Branch: [branch]
12. Push to Deploy: ON
13. Copy Deployment URL → Add as GitHub Webhook
    - Payload URL: [deployment URL from xCloud]
    - Content Type: application/json

--- FOR PRIVATE REPO ---
14. Copy Public Key from xCloud → Add as Deploy Key in GitHub
    - GitHub Repo → Settings → Deploy Keys → Add

--- DEPLOYMENT SCRIPT ---
15. Paste this script:
[prepared deployment script]

--- .ENV FILE (NODE.JS ONLY) ---
16. .env File Path: [path]
17. File Content:
[prepared .env content]

18. Click START to deploy!
========================================
```

---

## Phase 3: Panel Guidance (Interactive)

Walk the user through the panel step by step:

### Step 1: Direct them to xCloud Dashboard
Tell the user: **"Open your xCloud dashboard and click 'Add New Site'."**

### Step 2: Guide Through Each Screen
For each panel screen, tell them exactly what to select/enter using the cheat sheet.
Wait for confirmation before moving to the next step.

### Step 3: Critical Moments - Remind User to Copy

**After enabling Push-to-Deploy:**
"Copy the Deployment URL now! You need to add it as a webhook in GitHub."

**After Git Repo tab shows Public Key:**
"Copy this Public Key! Add it as a Deploy Key in your GitHub repo → Settings → Deploy Keys."

**After Database creation:**
"SAVE the database password! You won't see it again. Copy it now."

### Step 4: GitHub Webhook Setup

Guide user to set up webhook:
1. Go to GitHub Repo → Settings → Webhooks → Add webhook
2. Payload URL: [deployment URL from xCloud]
3. Content Type: `application/json`
4. Which events: Just the push event
5. Active: checked

### Step 5: Deploy Key (for private repos)

Guide user to add deploy key:
1. Go to GitHub Repo → Settings → Deploy Keys → Add deploy key
2. Title: "xCloud - [site-name]"
3. Key: [paste public key from xCloud]
4. Allow write access: optional

### Step 6: Start Deployment

Tell user to click **Start** and wait for deployment to complete.

---

## Phase 4: Post-Setup (Automated)

After the user confirms the site is created in xCloud:

### Step A: Get New Site Details

Ask user for:
- **Site URL** (staging or live)
- **Database password** (if they saved it, for connection string - DO NOT store it in files)
- **SSH details** for the new site (may differ from server SSH)

### Step B: Verify Site is Running

```bash
ssh -p [port] [user]@[host] "curl -s -o /dev/null -w '%{http_code}' http://localhost:[app-port]"
```

### Step C: Create/Update .xcloud/context.md

If the project already has xcloud-init context, update it.
If not, create `.xcloud/context.md` with full details (refer to xcloud-init.md format).

Add site-specific section:
```markdown
## Site Details
- **Site Type:** [Node.js SSR / Node.js CSR / WordPress]
- **Site URL:** [url]
- **xCloud Dashboard:** https://app.xcloud.host
- **Node.js Version:** [version] (if applicable)
- **Port:** [port] (if applicable)
- **PHP Version:** [version] (if applicable)
- **Push-to-Deploy:** Enabled on branch [branch]
- **Database:** [db-name] (MariaDB)
```

### Step D: Test Push-to-Deploy

Make a small test commit and push to verify auto-deploy works:
```bash
# Make a trivial change (add comment or update timestamp)
git add .
git commit -m "test: verify push-to-deploy"
git push origin [branch]
```

Then SSH to verify the deployment triggered.

### Step E: Summary

Present final summary:
```
========================================
 Site Created Successfully!
========================================
Site Type:    [type]
Site URL:     [url]
Server:       [host]
Database:     [db-name]
Push-to-Deploy: Active on [branch]

Available Commands:
  /xcloud-ssh    - SSH to this server
  /xcloud-db     - Query the database
  /xcloud-push   - Push changes & deploy
  /xcloud-pull   - Pull latest from GitHub
  /xcloud-status - Full status check
========================================
```

---

## Important Notes

- NEVER store database passwords in any file
- xCloud has no API - all site creation goes through https://app.xcloud.host
- WordPress Git Clone REQUIRES `wp-config-sample.php` in the repo
- Node.js apps need the correct Port and Start Command or they won't work
- xCloud supports NGINX and OpenLiteSpeed stacks - both support WordPress and Node.js
- Push-to-Deploy requires both: webhook in GitHub + deploy key (for private repos)
- After site creation, verify the deployment actually worked before considering it done
