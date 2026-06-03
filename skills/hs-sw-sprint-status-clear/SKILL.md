---
name: hs-sw-sprint-status-clear
description: Remove the sprint status bar from Claude Code — run after a sprint closes or when the status bar is stale
---

# /sprint-status-clear — Remove Sprint Status Bar

Removes the sprint status line from the project-level `.claude/settings.json` and deletes
`tmp/sprint-status.sh`. Use this when a sprint has closed and the status
bar is no longer needed.

## Process

### Step 1 — Remove from project-level settings.json

- Read `<project-root>/.claude/settings.json` (NOT `~/.claude/settings.json` — never modify global settings)
- If `statusLine` key is present: remove it
- Write back — preserve all other settings
- If file is now `{}` or only had `statusLine`: leave the file as `{}`
- If file doesn't exist or has no `statusLine`: report "Status bar was not configured in project settings" and stop
- Also check `~/.claude/settings.json` — if it has a `statusLine` pointing to this project's script, warn the user: "Found statusLine in global settings (~/.claude/settings.json) — remove it manually to prevent it bleeding into other sessions"

### Step 2 — Remove script

- Delete `tmp/sprint-status.sh` if it exists
- If it doesn't exist: skip silently

### Step 3 — Confirm

Report to user:
```
✓ Sprint status bar removed.
  • statusLine removed from .claude/settings.json
  • tmp/sprint-status.sh deleted
Reload Claude Code (Cmd+R) if the status bar is still visible.
```
