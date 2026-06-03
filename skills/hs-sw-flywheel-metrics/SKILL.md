---
name: hs-sw-flywheel-metrics
description: Show the rework / escape-rate trend across all sprints and projects — the gate that decides whether you can scale agent count
argument-hint: [--project <name>] [--last <N>]
---

# /flywheel-metrics — Rework Trend Across Sprints

Reads the global metrics log (`~/.claude/flywheel/sprint-metrics.jsonl`, written
one line per sprint by `/hs-sw-sprint-close`) and shows your **escape rate** trend.

## Why this exists

Yegge's diagnostic: *when >20% of agent output requires rework, the tasks are
poorly scoped — don't add more agents.* In this flywheel, "rework" = **escape
defects**: fix-beads created to repair work that was *already QA-passed*, counted
by the gate that caught them (`caught:review` / `caught:manual` / `caught:pr`).
First-pass QA fails are NOT rework — that's TDD working, reported separately as
`qa_fails`.

One sprint's number is noise. This view shows the **trend** across sessions and
projects — the only thing that tells you whether your scoping is actually
improving and whether you've earned the right to allocate more agents.

## Process

### Step 1 — Read the log

```bash
cat ~/.claude/flywheel/sprint-metrics.jsonl 2>/dev/null
```
If the file is missing or empty: report "No sprint metrics logged yet. Run a
sprint to completion and `/hs-sw-sprint-close` will record the first line." Stop.

### Step 2 — Filter (optional)

- `--project <name>`: only rows where `project` matches.
- `--last <N>`: only the most recent N rows (default: all, but cap display at 15).

### Step 3 — Compute and render

For the selected rows compute:
- **Trailing escape rate** = `sum(review+manual+pr) / sum(orig)` across the window
  (weighted by ticket count, not a mean of percentages).
- Per-project escape rate (same weighted formula, grouped by `project`).
- Direction: compare the trailing rate of the last 3 sprints vs the prior 3.

Render:
```
Rework / escape-rate trend  (last 8 sprints, all projects)

  trailing escape rate:  16%   ✓ under 20% — clear to scale agent count
  direction:             ↓ improving (last 3: 12% vs prior 3: 21%)

  date        project       sprint    orig  escape   qa_fails
  2026-06-02  jstm-inputs   GH#323     13    31% ⚠      1
  2026-05-28  jstm-svc-conn GH#301      9     0% ✓      2
  2026-05-22  jstm-gh-107   GH#269     24    29% ⚠      7
  ...

  by project:  jstm-inputs 18% · jstm-gh-107 24% ⚠ · jstm-svc-conn 6% ✓

  Verdict: <if trailing <20%> You may allocate more agents next sprint.
           <if trailing ≥20%> Hold agent count. Tickets are under-scoped —
           tighten /hs-sw-beads-create + Phase 0 enrichment before scaling.
```

### Step 4 — Flag outliers

- Mark any single sprint with escape rate ≥20% with `⚠`.
- If the same project is repeatedly ≥20%, call it out explicitly — that project's
  decomposition needs work, not more agents.

## Rules

- Read-only. Never modify the log.
- Weight by ticket count — a 50%-escape 2-ticket sprint shouldn't dominate a
  13-ticket sprint. Always use `sum(escapes)/sum(orig)`, never `mean(rates)`.
- The 20% threshold is Yegge's; state it as the scaling gate, not a quality grade.
