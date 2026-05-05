---
name: auto-sync-config
description: This skill is used when the user modifies Claude configuration files like .claude.json, settings.json, or any Claude-related settings. It automatically commits and pushes changes to the claude-config GitHub repository. Activate when user says "sync config", "update claude settings", "save changes to github", or when config files in ~/.claude are modified.
version: 1.0.0
---

# Auto Sync Config Skill

Automatically syncs Claude configuration changes to GitHub.

## When This Skill Applies

- User modifies `~/.claude.json` or other config files
- User says "sync config", "save to github", "push changes"
- After any settings modification in Claude

## Sync Operation

### Files to Sync
- `~/.claude.json` - Main configuration
- `~/.claude/history.jsonl` - Conversation history
- `~/.claude/settings.json` - Settings (if exists)
- `~/.claude/settings.local.json` - Local settings (if exists)

### Sync Process

1. Detect changed files in `~/.claude/`
2. Stage modified files
3. Create commit with descriptive message
4. Push to `git@github.com:yeyou20010101-sys/claude-config.git`

### Commit Message Format
```
Auto-sync: [timestamp] - [changed files]
```

### Excluded Files (Never Sync)
- `backups/` - Backup directory
- `cache/` - Cache files
- `sessions/` - Session info
- `shell-snapshots/` - Shell snapshots
- `plugins/` - Plugin directory
- `ide/` - IDE files
- `session-env/` - Session environment

## Commands

```bash
# Manual sync
cd ~/.claude && git add . && git commit -m "Auto-sync: $(date '+%Y-%m-%d %H:%M')" && git push

# Check status
cd ~/.claude && git status

# View recent commits
cd ~/.claude && git log --oneline -5
```

## Security Notes

- Never sync files containing tokens/passwords
- Filter sensitive data before sync (e.g., PAT tokens, API keys)
- Repository is private (claude-config) - settings visibility set to private