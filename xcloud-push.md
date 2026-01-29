---
allowed-tools: Bash(git:*), Bash(ssh:*), Read, Write, Glob, Grep
description: Commit changes and push to GitHub (may trigger auto-deploy if configured)
---

## Context

- Current directory: !`pwd`
- Git status: !`git status --short 2>/dev/null`
- Current branch: !`git branch --show-current 2>/dev/null`
- Unpushed commits: !`git log origin/$(git branch --show-current 2>/dev/null)..HEAD --oneline 2>/dev/null | head -5`
- Recent commits: !`git log --oneline -5 2>/dev/null`

## Your Task: Push Changes to GitHub

Commit local changes and push to GitHub.

### Step 1: Verify Repository State

1. Confirm in a git repository
2. Read `.xcloud/context.md` for project info
3. Check **Auto-Deploy** setting in context.md to determine if pushing will trigger deployment
4. Check current branch matches deploy configuration (if auto-deploy is enabled)

### Step 2: Review Changes

Show user:
1. Unstaged changes (modified files)
2. Staged changes (ready to commit)
3. Untracked files (new files)
4. Any unpushed commits

### Step 3: Stage and Commit

If there are uncommitted changes:

1. Show diff summary: `git diff --stat`
2. Use AskUserQuestion to ask what to commit:
   - **All Changes** - Stage everything and commit
   - **Select Files** - Let user specify which files
   - **Only Staged** - Commit only already-staged files
   - **Skip Commit** - Just push existing commits

3. Ask for commit message or auto-generate based on changes

4. Execute:
```bash
git add [files]
git commit -m "[message]"
```

### Step 4: Pre-Push Checklist

Before pushing, verify:
1. No sensitive files being committed (.env, credentials, etc.)
2. Branch is correct
3. No obvious errors in code (if detectable)

**If Auto-Deploy is enabled** (check context.md):
```
WARNING: Pushing to [branch] will trigger auto-deploy to LIVE site!
Continue? (User must confirm)
```

**If Auto-Deploy is NOT enabled**:
```
Ready to push to [branch].
Note: This will NOT auto-deploy. You'll need to manually deploy via SSH if needed.
Continue?
```

### Step 5: Execute Push

```bash
git push origin [branch]
```

If push fails:
- If rejected due to remote changes: suggest pull first
- If permission denied: check credentials
- If branch doesn't exist remotely: `git push -u origin [branch]`

### Step 6: Post-Push Actions

**If Auto-Deploy is enabled**:
1. Inform user that auto-deploy should trigger
2. Optionally SSH to server to verify deployment:
```bash
ssh -p [port] [user]@[host] "cd [remote-path] && git log -1 --oneline"
```

**If Auto-Deploy is NOT enabled**:
1. Inform user the push was successful
2. Ask if they want to manually deploy now via SSH:
   - If yes: SSH to server and run `git pull` in the remote path
   - If no: Just confirm push completed

### Step 7: Update Context

Add to Development Log in `.xcloud/context.md`:
```markdown
### [DATE] [TIME] - Git Push & Deploy
- Branch: [branch]
- Commit: [hash] - [message]
- Files changed: [list]
- Auto-deploy: [Yes - triggered / No - manual deploy needed]
- Status: [success/pending verification]
```

Add to Recent Deployments section:
```markdown
| Date | Commit | Changes | Status |
|------|--------|---------|--------|
| [DATE] | [short-hash] | [summary] | Deployed |
```

## Important Notes

- ALWAYS check Auto-Deploy setting in context.md before pushing
- If auto-deploy is enabled, ALWAYS warn before pushing to deploy branch
- Never force push without explicit user confirmation
- Check for .env or credential files before committing
- If deploy fails, user may need to SSH and check manually
- Consider running a quick syntax check before deploying
