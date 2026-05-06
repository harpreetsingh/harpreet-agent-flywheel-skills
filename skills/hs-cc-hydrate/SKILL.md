---
name: hs-cc-hydrate
description: Restore session context from the most recent stash after compaction or at the start of a new session.
argument-hint: [YYYY-MM-DD] [list]
---

# /hydrate — Restore session context

Restore context from a stash file in `tmp/stash/` (relative to project root).

## Parse arguments

| Argument | Example | Meaning |
|----------|---------|---------|
| *(none)* | `/hydrate` | Load the most recent stash file |
| `YYYY-MM-DD` | `/hydrate 2026-05-03` | Load the most recent stash from that date |
| `list` | `/hydrate list` | Show available stash files, don't load any |

## Steps

1. **Find stash files.** List `tmp/stash/*-session-resume.md` sorted by filename
   (newest first). Files are named `YYYY-MM-DD-HHMMSS-session-resume.md` so
   alphabetical = chronological.

   - If no stash files exist, also check for a legacy `tmp/session-resume.md`.
     If found, use it and suggest the user run `/stash` to migrate to the new format.
   - If nothing exists at all: "No stash found. Use `/stash` to create one." and stop.

2. **Select the file.**
   - `list` argument: show all available stashes as a table (`date | time | first line`)
     and stop. Don't load anything.
   - Date argument: find the most recent stash file matching that date prefix. If no
     match: "No stash found for YYYY-MM-DD. Run `/hydrate list` to see what's available."
   - No argument: pick the most recent file.

3. **Read the selected stash file.**

4. Run `git status` and `git log --oneline -3` to get current branch state,
   uncommitted changes, and how far ahead/behind remote.

5. Present a brief summary combining both sources:
   - Where we were (from stash)
   - Current git state (branch, any drift since stash)
   - What the immediate next step is
   - How many open threads exist
   - Which stash file was loaded (path + date)

6. Ask: "Ready to pick up from here, or do you want to adjust the plan?"

## Rules

- Do NOT re-read every file listed in the resume upfront. Only read files as needed when you actually start working.
- Do NOT treat the resume as a task list to execute automatically. It's context for the human to direct you.
- The resume was written by a previous Claude instance. Trust it but verify if something seems off.
- If git state has diverged from what the stash recorded (different branch, new commits from others), flag this explicitly.
