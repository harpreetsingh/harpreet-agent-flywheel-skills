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

### Step 3 — Compute the rework (escape) metric

This is the number that tells the *next* sprint whether it can safely allocate
more agents (Yegge's <20% rule). It is computed from durable bead labels — no
session state required.

**Denominator** — `orig` = count of sprint beads from Step 1 that are NOT
themselves fix-beads (exclude any bead carrying a `caught:*` label).

**Escape defects** — repair work filed *after* a bead was already qa-passed.
Count beads (sprint-linked or created during the sprint window) by catch-gate:
```bash
bd list --label caught:review --json   # found by wave-gate review agents
bd list --label caught:manual --json   # human rejected a qa-passed ticket
bd list --label caught:pr --json       # CI red / change-request after PR opened
```
Filter each to this sprint (same feature label, or created after the sprint's
first bead). Let `review`, `manual`, `pr` be the counts.

**Cheap signal (reported, NOT in escape rate)** — `qa_fails` = count of sprint
beads with a `[QA-FAIL]` comment (first-pass QA fails = TDD working).

**Escape rate** = `(review + manual + pr) / orig`.

Show it:
```
Rework / escape metric for this sprint:
  orig tickets:        13
  caught:review         3   (escaped QA, found at wave gate)
  caught:manual         0   (escaped to human review)
  caught:pr             1   (escaped to CI/PR — most expensive)
  ─────────────────────────
  escape rate:         31%   ⚠ over 20% — do NOT add agents next sprint
  qa_fails (cheap):     1    (first-pass, TDD working — informational)
```

### Step 4 — Dry-run check

If `$ARGUMENTS` contains `--dry-run`:
```
Dry-run: would close 13 beads, log escape rate 31% to the metrics file,
and remove the status bar. Run without --dry-run to proceed.
```
Stop here (do NOT append to the log or close beads in dry-run).

### Step 5 — Close beads

For each qa-passed bead in the sprint: `bd close <id>`

Report each one as it closes. If any fail, note them and continue — don't abort.

After closing: show count of successfully closed vs failed.

### Step 6 — Append metrics to the global log

Append exactly one line to `~/.claude/flywheel/sprint-metrics.jsonl` (create the
`~/.claude/flywheel/` directory if missing). This is append-only — never rewrite
existing lines. Use the project's directory name as `project` and the sprint's
GH/epic id as `sprint`:
```bash
mkdir -p ~/.claude/flywheel
python3 - <<'PY'
import json, os, datetime, pathlib
rec = {
    "date": datetime.date.today().isoformat(),
    "project": pathlib.Path(os.getcwd()).name,
    "sprint": "<GH#/epic id>",
    "orig": 13,
    "caught": {"review": 3, "manual": 0, "pr": 1},
    "escape_rate": round((3 + 0 + 1) / 13, 3),
    "qa_fails": 1,
}
log = pathlib.Path.home() / ".claude" / "flywheel" / "sprint-metrics.jsonl"
with log.open("a") as f:
    f.write(json.dumps(rec) + "\n")
print("logged:", rec)
PY
```
Substitute the real counts computed in Step 3. View the trend any time with
`/hs-sw-flywheel-metrics`.

### Step 7 — Remove status bar

- Read `<project-root>/.claude/settings.json` (NOT `~/.claude/settings.json` — never modify global settings)
- Remove the `statusLine` key if present
- Write back, preserving all other settings
- Also check `~/.claude/settings.json` — if it has a `statusLine` pointing to this project's script, warn the user to remove it manually

### Step 8 — Delete tmp files

Delete these files if they exist (skip silently if missing):
- `tmp/sprint-status.sh`
- `tmp/sprint-exec-plan.md`

### Step 9 — Report

```
✓ Sprint closed.
  • 13 beads closed
  • Escape rate 31% logged to ~/.claude/flywheel/sprint-metrics.jsonl
  • Status bar removed from .claude/settings.json
  • tmp/sprint-status.sh deleted
  • tmp/sprint-exec-plan.md deleted

Sprint state checkpoint lives at: <feature_dir>/sprint-state.md
Trend across all sprints: /hs-sw-flywheel-metrics
Reload Claude Code (Cmd+R) to clear the status bar from the UI.
```

If any beads failed to close, list them:
```
⚠ 2 beads failed to close — retry manually:
  bd close joystream-xxxx
  bd close joystream-yyyy
```
