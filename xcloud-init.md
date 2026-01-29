---
allowed-tools: Bash(git clone:*), Bash(mkdir:*), Bash(cd:*), Bash(ls:*), Bash(ssh:*), Write, Read, Glob, Grep
description: Initialize a new xCloud project - clone repo, setup context folder, and gather project configuration
---

## Your Task: Initialize xCloud Project

You are initializing a new xCloud hosting project. This involves:
1. Gathering project configuration
2. Cloning the GitHub repository
3. Creating the context folder and project documentation
4. Testing connections

## Step 1: Gather Project Information

Ask the user for the following information using AskUserQuestion for structured choices where applicable:

### Required Information:
1. **Project Name** - A short identifier for this project (e.g., "my-website")
2. **Local Path** - Where to clone the project locally (e.g., "C:\dev\projects\my-website")
3. **GitHub Repository URL** - The full repo URL (SSH or HTTPS)
4. **GitHub Branch** - Main branch name (usually "main" or "master")
5. **Auto-Deploy Enabled** - Does pushing to GitHub automatically deploy to the server? (Yes/No)
6. **Deploy Branch** - If auto-deploy is enabled, which branch triggers it? (often same as main)

### SSH Connection Details:
7. **SSH Host** - Server hostname or IP
8. **SSH Port** - Usually 22, but xCloud may use custom ports
9. **SSH Username** - Login user
10. **Remote Project Path** - Path on server (e.g., "/home/user/public_html/")

### SSH Key Authentication (Recommended):
After gathering SSH details, check if the user has set up SSH key authentication for this server:

**Ask:** "Have you added your SSH public key to this server for passwordless authentication?"
- **Yes** - Great, connections will be passwordless
- **No, help me set it up** - Guide them through the process (see SSH Key Setup section below)
- **No, I'll use password** - That's fine, they'll be prompted for password each time

### Database Details:
11. **DB Host** - Database server (often "localhost" if same server)
12. **DB Port** - Usually 3306 for MariaDB
13. **DB Name** - Database name
14. **DB Username** - Database user
15. **DB Requires SSH Tunnel** - Yes/No (if DB is not publicly accessible)

## SSH Key Setup Guide

If the user needs help setting up SSH key authentication, guide them through this:

### How SSH Keys Work (Brief Explanation)
```
Your PC has: Private Key (secret, stays here) + Public Key (share this)
Server has: Your Public Key in authorized_keys
When connecting: Your PC proves it has the private key → Server grants access
```

### Step-by-Step:
1. **Check for existing local key:**
   ```bash
   ls ~/.ssh/id_ed25519.pub 2>/dev/null || ls ~/.ssh/id_rsa.pub 2>/dev/null
   ```

2. **If no key exists, generate one:**
   ```bash
   ssh-keygen -t ed25519 -C "your-email@example.com"
   ```
   (Accept defaults, optionally set a passphrase)

3. **Display the public key:**
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

4. **Add to server:** The user needs to add this public key to their server's SSH authentication settings. In xCloud control panel, this is typically under:
   - SSH Access → Public Keys, or
   - Security → SSH Keys

   They paste the entire public key line (starts with `ssh-ed25519` or `ssh-rsa`).

5. **Test the connection:**
   ```bash
   ssh -p [port] [user]@[host]
   ```
   If successful, it should connect without asking for a password.

### Important Notes:
- One key works for ALL servers - just add the same public key to each server
- The private key NEVER leaves your machine
- If they already set up a key for another project on the same server, it will already work

---

## Step 2: Clone Repository

After gathering info, clone the repository:
```bash
git clone <repo-url> <local-path>
```

## Step 3: Create Context Folder

Create a `.xcloud` folder inside the project with:
- `context.md` - Main context file tracking all project info
- Store configuration (without passwords) for reference

## Step 4: Generate Context File

Create `.xcloud/context.md` with this structure:

```markdown
# Project: [PROJECT_NAME]

## Quick Reference
- **Local Path:** [path]
- **GitHub:** [repo-url]
- **Main Branch:** [branch]
- **Auto-Deploy:** [yes/no]
- **Deploy Branch:** [deploy-branch or N/A if no auto-deploy]
- **Server:** [ssh-user]@[ssh-host]:[ssh-port]
- **Remote Path:** [remote-path]
- **Database:** [db-name] on [db-host]:[db-port]

## Connection Commands
```bash
# SSH into server
ssh -p [port] [user]@[host]

# SSH tunnel for database (if needed)
ssh -p [port] -L 3307:[db-host]:[db-port] [user]@[host]
```

## SSH Key Status
- **Key configured:** [yes/no]
- If no: Password will be prompted for each connection

## Development Log
<!-- Auto-updated by Claude - newest entries at top -->

### [DATE] - Project Initialized
- Repository cloned
- Context folder created
- Initial setup complete

---

## Database Schema
<!-- Will be populated when /xcloud-db schema is run -->

## Recent Deployments
<!-- Auto-updated when pushes are made -->

## Notes
<!-- Add project-specific notes here -->
```

## Step 5: Verify Setup

1. Confirm the repository was cloned successfully
2. Confirm the context folder was created
3. Optionally test SSH connection (ask user if they want to test now)

## Important Notes

- NEVER store passwords in any file
- If SSH key is not set up, passwords will be prompted for each connection
- The context.md file should be git-ignored (add to .gitignore if not already)
- Update the context.md with today's date and initialization info

## Output

After completion, summarize:
- Project location
- SSH key status (configured or not)
- Commands available: /xcloud-ssh, /xcloud-db, /xcloud-pull, /xcloud-push, /xcloud-status
- If SSH key not configured: remind user they can set it up later for passwordless access
