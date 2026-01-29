---
allowed-tools: Bash(ssh:*), Read, Glob, Write
description: Connect to xCloud server via SSH and execute commands
---

## Context

- Current directory: !`pwd`
- Looking for xCloud context file...

## Your Task: SSH Connection to xCloud Server

You need to establish an SSH connection to the xCloud server for this project.

### Step 1: Find Project Context

First, locate the `.xcloud/context.md` file in the current directory or parent directories:
1. Check current directory for `.xcloud/context.md`
2. If not found, check parent directories
3. If no context file found, inform user to run `/xcloud-init` first

### Step 2: Read Connection Details

From the context file, extract:
- SSH Host
- SSH Port
- SSH Username
- Remote Project Path

### Step 3: Ask What User Wants to Do

Use AskUserQuestion to ask what they want to do:
- **Interactive Shell** - Open an interactive SSH session
- **Run Command** - Execute a specific command on the server
- **Check Server Status** - Run diagnostic commands (disk space, memory, processes)
- **View Logs** - Check recent error/access logs
- **Restart Services** - Restart web server or PHP

### Step 4: Execute SSH Command

**For Interactive Shell:**
```bash
ssh -p [port] [user]@[host]
```
Note: This will prompt for password. The user will interact directly.

**For Run Command:**
Ask user for the command, then:
```bash
ssh -p [port] [user]@[host] "[command]"
```

**For Server Status:**
```bash
ssh -p [port] [user]@[host] "df -h && free -m && ps aux | head -20"
```

**For View Logs:**
```bash
ssh -p [port] [user]@[host] "tail -100 ~/logs/error.log"
```

### Step 5: Update Context

After any operation, add an entry to the Development Log section in `.xcloud/context.md`:
```markdown
### [DATE] [TIME] - SSH Session
- Action: [what was done]
- Result: [outcome]
```

## Important Notes

- Password is prompted by SSH directly - never ask for or store it
- If connection fails, suggest checking:
  - SSH credentials
  - Firewall rules
  - Server status
- For long-running commands, consider using `nohup` or `screen`
