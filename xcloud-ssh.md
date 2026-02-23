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

### Step 3: Check SSH Key Auth Status

Before presenting options, check if SSH key auth is already working:
```bash
ssh -o ConnectTimeout=5 -o BatchMode=yes -p [port] [user]@[host] "echo SSH_KEY_OK" 2>/dev/null
```
- If output contains `SSH_KEY_OK` → key auth is working, proceed normally
- If it fails → key auth is NOT set up, add "Setup SSH Key Auth" as the first option

### Step 4: Ask What User Wants to Do

Use AskUserQuestion to ask what they want to do:
- **Setup SSH Key Auth** (only show if key auth is not working) - One-time setup for passwordless SSH
- **Interactive Shell** - Open an interactive SSH session
- **Run Command** - Execute a specific command on the server
- **Check Server Status** - Run diagnostic commands (disk space, memory, processes)
- **View Logs** - Check recent error/access logs
- **Restart Services** - Restart web server or PHP

### Step 5: Execute SSH Command

**For Setup SSH Key Auth:**

This is a one-time setup. After this, all SSH connections will be passwordless.

1. Find the user's local public key:
   ```bash
   cat ~/.ssh/id_ed25519.pub 2>/dev/null || cat ~/.ssh/id_rsa.pub 2>/dev/null
   ```

2. If no key exists, generate one first:
   ```bash
   ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)" -N "" -f ~/.ssh/id_ed25519
   cat ~/.ssh/id_ed25519.pub
   ```

3. Push the public key to the server in a single command (user enters password ONCE):
   ```bash
   ssh -p [port] [user]@[host] "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo '[PUBLIC_KEY_CONTENT]' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo 'SSH key added successfully'"
   ```
   IMPORTANT: Replace `[PUBLIC_KEY_CONTENT]` with the actual content of the public key file (the full line starting with `ssh-ed25519` or `ssh-rsa`).

4. Verify it works (should connect without password):
   ```bash
   ssh -o BatchMode=yes -p [port] [user]@[host] "echo 'Passwordless SSH is working!'"
   ```

5. Update `.xcloud/context.md` SSH Key Status to `yes`

**For Interactive Shell:**
```bash
ssh -p [port] [user]@[host]
```

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

### Step 6: Update Context

After any operation, add an entry to the Development Log section in `.xcloud/context.md`:
```markdown
### [DATE] [TIME] - SSH Session
- Action: [what was done]
- Result: [outcome]
```

## Important Notes

- **SSH Key Auth preferred**: Always check key auth status first. If not set up, proactively suggest it as the first option - it's a one-time setup that eliminates all future password prompts.
- Never store passwords in any file
- If connection fails, suggest checking:
  - SSH credentials
  - Firewall rules (fail2ban may block IP after failed attempts)
  - Server status
- For long-running commands, consider using `nohup` or `screen`
- On Windows, `ssh-copy-id` is NOT available. Use the manual method (push key via SSH command) instead.
