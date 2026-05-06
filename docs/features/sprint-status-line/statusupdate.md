# Sprint Status Line for Claude Code

## Problem

During a multi-agent sprint, the user has no visibility into progress without manually running `bd show` or asking for a status update. They want a live progress indicator in the Claude Code status bar.

## UX

The status line displays in Claude Code's status bar (above the input panel):

```
Sprint: ■■▣□□ 2/5 done · W1:✓ W2:◐ W3:○ W4:○
```

### Symbols

| Symbol | Meaning |
|--------|---------|
| ■ | Ticket closed |
| ▣ | Ticket in progress |
| □ | Ticket open (not started) |
| ✓ | Wave complete (all tickets closed) |
| ◐ | Wave in progress (at least one ticket active or closed) |
| ○ | Wave not started |

### Lifecycle

As the sprint progresses:
```
Start:      Sprint: □□□□□ 0/5 done · W1:○ W2:○ W3:○ W4:○
Wave 1 WIP: Sprint: ▣▣□□□ 0/5 done · W1:◐ W2:○ W3:○ W4:○
Wave 1 done:Sprint: ■■□□□ 2/5 done · W1:✓ W2:○ W3:○ W4:○
Wave 2 WIP: Sprint: ■■▣□□ 2/5 done · W1:✓ W2:◐ W3:○ W4:○
All done:   Sprint: ■■■■■ 5/5 done · W1:✓ W2:✓ W3:✓ W4:✓
```

## Configuration

### Claude Code settings.json

Add to the project-level `.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash /path/to/sprint-status.sh"
  }
}
```

The status line refreshes automatically on Claude Code's internal cadence.

### Status Script

The script reads ticket status from beads (`bd`) and formats the output. It's designed to be fast (single `bd list` call piped to python) and silent on errors.

**Key design decisions:**
- Uses `bd show <all-ids> --json` to fetch all sprint tickets in one call (IDs are hardcoded per sprint)
- Wave-to-ticket mapping is hardcoded in the script (generated per sprint by the sprint-exec-plan step)
- A ticket is "done" when it has the `qa-passed` label (agents never close beads — humans do after review)
- Falls back to "Sprint: loading..." if bd or python fails
- Excludes the epic itself (only counts implementation tickets)

```bash
#!/usr/bin/env bash
# Sprint status line for Claude Code
# Reads tickets by ID and formats wave progress

cd <abs-path> 2>/dev/null || true

ALL_IDS="<space-separated ids>"

bd show $ALL_IDS --json 2>/dev/null | python3 -c '
import sys, json

try:
    data = json.load(sys.stdin)
except Exception:
    print("Sprint: loading...")
    sys.exit(0)

# Build status map: "done" = has qa-passed label (agents never close beads)
statuses = {}
for item in data:
    if "id" not in item:
        continue
    labels = item.get("labels", [])
    if "qa-passed" in labels:
        statuses[item["id"]] = "done"
    else:
        statuses[item["id"]] = item.get("status", "open")

# Wave-to-ticket mapping — generated per sprint
WAVES = [
    ["bead-id-1", "bead-id-2"],   # Wave 1
    ["bead-id-3"],                  # Wave 2
    ["bead-id-4"],                  # Wave 3
    ["bead-id-5"],                  # Wave 4
]

def wave_symbol(ids):
    stats = [statuses.get(i, "open") for i in ids]
    if all(s == "done" for s in stats):
        return "\u2713"   # checkmark
    if any(s in ("in_progress", "done") for s in stats):
        return "\u25d0"   # half circle
    return "\u25cb"       # empty circle

all_ids = [i for w in WAVES for i in w]
total = len(all_ids)
done = sum(1 for i in all_ids if statuses.get(i) == "done")

bar = ""
for i in all_ids:
    s = statuses.get(i, "open")
    if s == "done":
        bar += "\u25a0"   # filled square
    elif s == "in_progress":
        bar += "\u25a3"   # square with dot
    else:
        bar += "\u25a1"   # empty square

wave_parts = " ".join(
    "W{}:{}".format(n + 1, wave_symbol(w)) for n, w in enumerate(WAVES)
)

print("Sprint: {} {}/{} done \u00b7 {}".format(bar, done, total, wave_parts))
' || echo "Sprint: loading..."
```

## Integration with Sprint Workflow

### Where this fits in the flywheel

```
SHAPE → PLAN → REVIEW → DECOMPOSE → SPRINT PLAN → ★EXECUTE → CLOSE
                                         ↑
                         sprint-exec-plan generates the wave mapping
                         sprint-go configures the status line
```

### Automation opportunity

The `/hs-sw-sprint-exec-plan` skill already knows the wave-to-ticket mapping. It should:
1. Generate `tmp/sprint-status.sh` with the correct wave mapping
2. Configure `.claude/settings.json` with the status line command

The `/hs-sw-sprint-go` skill should verify the status line is configured before launching the Director.

### Cleanup

After a sprint completes, remove the status line:
- Run `/sprint-status-clear` to remove from settings.json and delete the script

## Future Enhancements

- **Generic sprint status skill** — `/hs-sw-sprint-status` that auto-generates the script from any epic's beads
- **Time tracking** — show elapsed time: `Sprint: ■■▣□□ 2/5 · 47m elapsed`
- **ETA** — based on avg ticket completion time: `Sprint: ■■▣□□ 2/5 · ~35m remaining`
- **Multi-sprint** — support multiple concurrent sprints with labels
- **Color support** — if Claude Code status line supports ANSI colors, use green/yellow/grey instead of Unicode symbols
