---
name: hs-sw-sprint-close
description: Wipe a completed sprint — close all qa-passed beads, remove status bar, clean up tmp files
argument-hint: [--dry-run]
---

# /sprint-close — Close a Completed Sprint

Full cleanup after a sprint is done: closes all qa-passed beads, removes the
status bar, and deletes sprint tmp files.

Use `--dry-run` to preview what would be closed without making changes.

## Process

### Step 1 — Find sprint ticket IDs

Two sources, in order:
1. Read `tmp/sprint-status.sh` — extract `ALL_IDS` from it
2. If not found, read `tmp/sprint-exec-plan.md` — extract ticket IDs from wave tables

If neither exists: report "No sprint plan found — nothing to close." and stop.

### Step 2 — Find qa-passed beads

Run: `bd list --label qa-passed --json`

Cross-reference against the sprint ticket IDs from Step 1 to get the sprint's
qa-passed beads (filter out any qa-passed beads from other sprints).

If zero qa-passed beads in this sprint: report "No qa-passed beads found —
nothing to close. Is the sprint actually complete?" and stop.

Show a summary table:
```
Sprint beads to close:
  joystream-m59b  Add LLM-driven execute_loop entrypoint       [qa-passed]
  joystream-0cht  Wire execute_loop into CLI                    [qa-passed]
  ...
  Total: 13 beads
```

### Step 3 — Dry-run check

If `$ARGUMENTS` contains `--dry-run`:
```
Dry-run: would close 13 beads and remove the status bar.
Run without --dry-run to proceed.
```
Stop here.

### Step 4 — Close beads

For each qa-passed bead in the sprint: `bd close <id>`

Report each one as it closes. If any fail, note them and continue — don't abort.

After closing: show count of successfully closed vs failed.

### Step 5 — Remove status bar

- Read `<project-root>/.claude/settings.json` (NOT `~/.claude/settings.json` — never modify global settings)
- Remove the `statusLine` key if present
- Write back, preserving all other settings
- Also check `~/.claude/settings.json` — if it has a `statusLine` pointing to this project's script, warn the user to remove it manually

### Step 6 — Delete tmp files

Delete these files if they exist (skip silently if missing):
- `tmp/sprint-status.sh`
- `tmp/sprint-exec-plan.md`

### Step 7 — Report

```
✓ Sprint closed.
  • 13 beads closed
  • Status bar removed from .claude/settings.json
  • tmp/sprint-status.sh deleted
  • tmp/sprint-exec-plan.md deleted

Sprint state checkpoint lives at: <feature_dir>/sprint-state.md
Reload Claude Code (Cmd+R) to clear the status bar from the UI.
```

If any beads failed to close, list them:
```
⚠ 2 beads failed to close — retry manually:
  bd close joystream-xxxx
  bd close joystream-yyyy
```
