---
allowed-tools: Bash(git:*), Read, Write, Glob
description: Pull latest changes from GitHub repository
---

## Context

- Current directory: !`pwd`
- Git status: !`git status 2>/dev/null || echo "Not a git repository"`
- Current branch: !`git branch --show-current 2>/dev/null`
- Remote info: !`git remote -v 2>/dev/null | head -2`

## Your Task: Pull Latest from GitHub

Sync local repository with remote GitHub repository.

### Step 1: Verify Git Repository

Check if current directory is within the project:
1. Verify `.git` folder exists (current or parent)
2. Locate `.xcloud/context.md` for project info
3. If not in a git repo, inform user

### Step 2: Check Current State

Before pulling:
1. Check for uncommitted changes: `git status`
2. Check current branch
3. Check if there are unpushed commits

### Step 3: Handle Uncommitted Changes

If there are uncommitted changes, use AskUserQuestion:
- **Stash Changes** - Stash, pull, then unstash
- **Commit First** - Commit changes before pulling
- **Discard Changes** - Reset and pull (WARNING: loses changes)
- **Cancel** - Abort the pull operation

### Step 4: Execute Pull

```bash
git fetch origin
git pull origin [branch]
```

If merge conflicts occur:
1. List conflicted files
2. Explain the conflicts
3. Ask user how to proceed:
   - Manually resolve
   - Accept theirs (remote version)
   - Accept ours (local version)
   - Abort merge

### Step 5: Post-Pull Tasks

After successful pull:
1. Show summary of changes pulled
2. Check if dependencies need updating (package.json, composer.json, etc.)
3. Remind about any migration scripts if applicable

### Step 6: Update Context

Add to Development Log in `.xcloud/context.md`:
```markdown
### [DATE] [TIME] - Git Pull
- Branch: [branch]
- Commits pulled: [count]
- Changes: [summary]
- Conflicts: [none/resolved/pending]
```

## Important Notes

- Always check for uncommitted work before pulling
- If using SSH for git, ensure SSH key is configured
- For HTTPS repos, credentials may be prompted by git
- Consider running `/xcloud-status` after pull to verify state
