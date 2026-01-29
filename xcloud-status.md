---
allowed-tools: Bash(git:*), Bash(ssh:*), Read, Write, Glob
description: Show comprehensive status of xCloud project - local, remote, and database
---

## Context

- Current directory: !`pwd`
- Git status: !`git status --short 2>/dev/null | head -20`
- Current branch: !`git branch --show-current 2>/dev/null`
- Last local commit: !`git log -1 --format="%h %s (%cr)" 2>/dev/null`

## Your Task: Show Project Status

Display comprehensive status of the xCloud project across all systems.

### Step 1: Find Project Context

Locate and read `.xcloud/context.md` for project configuration.
If not found, inform user to run `/xcloud-init` first.

### Step 2: Gather Status Information

Collect status from multiple sources:

**Local Repository:**
```bash
git status
git log -5 --oneline
git branch -vv
```

**Remote Comparison:**
```bash
git fetch origin
git log HEAD..origin/[branch] --oneline  # Commits to pull
git log origin/[branch]..HEAD --oneline  # Commits to push
```

**Server Status (via SSH):**
```bash
ssh -p [port] [user]@[host] "cd [path] && git log -1 --format='%h %s (%cr)' && echo '---' && ls -la | head -10"
```
Note: This will prompt for password

### Step 3: Display Status Report

Format output as:
```
=====================================
PROJECT STATUS: [project-name]
=====================================

LOCAL REPOSITORY
----------------
Branch: [branch]
Status: [clean/dirty]
Uncommitted changes: [count]
Last commit: [hash] - [message] ([time])

REMOTE (GitHub)
---------------
Commits ahead: [count]
Commits behind: [count]
Last remote commit: [hash] - [message]

SERVER (xCloud)
---------------
[If SSH succeeds]
Last deployed: [hash] - [message] ([time])
Sync status: [in sync / behind by X commits]

[If SSH fails]
Unable to connect - check credentials

DATABASE
--------
Connection: [status if tested]
Last query: [from context log]

RECENT ACTIVITY
---------------
[Last 5 entries from Development Log]

=====================================
```

### Step 4: Identify Issues

Check for and report:
- Uncommitted local changes
- Unpushed commits
- Server out of sync with GitHub
- Merge conflicts
- Detached HEAD state

### Step 5: Suggest Actions

Based on status, suggest next steps:
- "Run /xcloud-push to deploy X commits"
- "Run /xcloud-pull to get X new commits"
- "Uncommitted changes - commit or stash before pulling"
- "Server is behind - check auto-deploy status"

### Step 6: Update Context

Add to Development Log:
```markdown
### [DATE] [TIME] - Status Check
- Local: [clean/X uncommitted]
- Remote sync: [ahead X/behind X/in sync]
- Server sync: [in sync/behind X/unknown]
```

## Important Notes

- SSH connection will prompt for password
- If user doesn't want to test SSH, skip server status
- Use AskUserQuestion to ask if they want full status (including SSH) or quick status (local only)
- Keep status output concise but informative
