---
name: project-sync
description: Used when user mentions syncing projects between local ~/01_Projects folder and GitHub. Handles git clone/pull from GitHub to local and git commit/push from local to GitHub. Triggers when user says 'sync project', 'pull project', 'push project', 'save to github', 'sync changes', or when project folder operations are needed.
version: 1.0.0
---

# Project Sync Skill

Manages project synchronization between local `~/01_Projects/` folder and GitHub repositories.

## Core Principle

- **Storage constrained**: Mac has limited disk space
- **Workflow**: Pull from GitHub → Work locally → Push back to GitHub when done
- **Confirmation required**: Always confirm before pushing changes to GitHub

---

## Commands

### 1. Clone/Pull Project from GitHub

```bash
# Clone new project to ~/01_Projects/
cd ~/01_Projects
git clone git@github.com:OWNER/REPO.git

# Pull latest changes for existing project
cd ~/01_Projects/PROJECT_NAME
git pull origin main
```

### 2. Push Project to GitHub (with confirmation)

**Before pushing, always ask user to confirm:**
```
⚠️ Confirm Push
Repository: [REPO_NAME]
Changes:
- [file1.txt] modified
- [file2.txt] added
- [file3.txt] deleted

Push to GitHub? (yes/no)
```

**If confirmed:**
```bash
cd ~/01_Projects/PROJECT_NAME
git add .
git status  # Show what will be committed
git commit -m "[description]"
git push origin main
```

**If rejected:** Do not push, inform user changes remain local.

---

## Workflow Examples

### Scenario: User wants to work on a project from another device

1. Ask for repository URL or name
2. Check if project exists locally in `~/01_Projects/`
3. If exists → `git pull` to get latest
4. If not exists → `git clone` to local
5. Report status to user

### Scenario: User finished work and wants to sync

1. List changed files: `git status`
2. Show changes summary to user
3. Request confirmation: "Push to GitHub?"
4. If yes → commit and push
5. If no → notify "Changes kept locally"

---

## Storage Management Tips

- **Before cloning**: Check local disk space with `df -h ~/`
- **Large repositories**: Consider shallow clone with `--depth 1`
- **After push**: Notify user they can delete local copy if needed

---

## Confirmation Template

```
=== Git Push Confirmation ===

Repository: OWNER/REPO_NAME
Location: ~/01_Projects/PROJECT_NAME

Changed Files:
[M] file1.txt - modified
[A] file2.txt - new
[D] file3.txt - deleted

Untracked: [count] files

Commit message: [auto-generated or custom]

⚠️ Push to GitHub? (yes/no)
```

---

## Safety Rules

1. **Never auto-push**: Always confirm with user first
2. **Show diff**: Display what changed before asking for confirmation
3. **Check branch**: Ensure on correct branch before push
4. **Warn on force**: If force push needed, get explicit confirmation
5. **Preserve history**: Never amend pushed commits without permission