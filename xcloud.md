---
allowed-tools: Bash(git:*), Read, Write, Glob, AskUserQuestion
description: xCloud project management hub - choose an action for your hosting project
---

## Context

- Current directory: !`pwd`
- xCloud context exists: !`test -f .xcloud/context.md && echo "Yes - Project initialized" || echo "No - Run /xcloud-init first"`
- Git status: !`git status --short 2>/dev/null | head -5 || echo "Not a git repository"`

## Your Task: xCloud Project Hub

You are the main entry point for xCloud project management. Help the user choose what they want to do.

### Step 1: Check Project Status

First, check if this directory has an xCloud project initialized:
1. Look for `.xcloud/context.md`
2. If found, read and display quick summary
3. If not found, suggest running `/xcloud-init`

### Step 2: Present Options

If project is initialized, use AskUserQuestion to offer:

- **Check Status** (/xcloud-status) - See local, remote, and server status
- **SSH to Server** (/xcloud-ssh) - Connect to xCloud hosting
- **Database Query** (/xcloud-db) - Query MariaDB database
- **Pull Changes** (/xcloud-pull) - Pull latest from GitHub
- **Push & Deploy** (/xcloud-push) - Commit, push, and deploy to live
- **View Context** - Read the full context.md file
- **Update Context** - Add notes to the context file

If project is NOT initialized:
- **Initialize Project** (/xcloud-init) - Set up a new xCloud project

### Step 3: Route to Appropriate Command

Based on user selection:
- Inform them of the specific command to run
- Or execute quick actions directly (View Context, Update Context)

### Quick Actions

**View Context:**
Read and display `.xcloud/context.md` in a formatted way.

**Update Context:**
Ask user for notes to add, then append to the Notes section:
```markdown
### [DATE] - User Note
[User's note here]
```

## Project Summary Display

If context exists, show:
```
xCloud Project: [name]
---
GitHub: [repo]
Server: [user]@[host]
Database: [db-name]
Local changes: [X files modified]
Last activity: [from log]
```

## Available Commands

Remind user of available commands:
- `/xcloud-init` - Initialize new project
- `/xcloud-status` - Full status check
- `/xcloud-ssh` - SSH to server
- `/xcloud-db` - Database operations
- `/xcloud-pull` - Pull from GitHub
- `/xcloud-push` - Push and deploy

## Notes

- All commands prompt for passwords when needed
- Context is stored in `.xcloud/context.md`
- Consider adding `.xcloud/` to `.gitignore` for sensitive project notes
