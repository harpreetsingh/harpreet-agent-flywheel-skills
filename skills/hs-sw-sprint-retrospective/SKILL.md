---
name: hs-sw-sprint-retrospective
description: Turn a finished sprint's failure data into patterns and concrete Phase 0 / AGENTS.md fixes — the actuator that drives escape rate down toward the 20% scaling gate
argument-hint: [feature-dir] [--sprint <gh#/epic-id>]
---

# /sprint-retrospective — Close the Learning Loop

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → ★CLOSE  │
│ ★ YOU ARE HERE: After the sprint, before the next one. Learn, then fix. │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

The escape-rate metric (`/hs-sw-flywheel-metrics`) **senses** whether rework is
under Yegge's 20% scaling threshold. This skill is the **actuator**: it reads the
sprint's failure data, finds recurring patterns, and proposes concrete changes to
Phase 0 enrichment / AGENTS.md / `beads-create` so the *next* sprint's escape rate
is lower. Sense → diagnose → correct → re-sense. Without this loop, a high escape
rate just persists.

Run it after `/hs-sw-sprint-close` (the failure data is durable in the beads DB and
the metrics log — it survives the sprint session).

## Data sources (all durable — created during the sprint)

1. **First-pass QA fails (cheap signal)** — beads with a `[QA-FAIL] <reason>` comment.
   These are TDD working: caught before `qa-passed`. The *reasons* reveal what
   workers get wrong on the first try.
   ```bash
   bd show <sprint-bead-ids> --json   # scan comments for "[QA-FAIL]"
   ```
2. **Escape defects (expensive signal)** — fix-beads labeled by the gate that caught
   them. Later gate = more expensive = worse.
   ```bash
   bd list --label caught:review --json   # escaped QA, caught at wave gate
   bd list --label caught:manual --json   # escaped to human review
   bd list --label caught:pr --json       # escaped to CI/PR — most expensive
   ```
   Filter to this sprint (feature label, or created after the sprint's first bead).
3. **The escape-rate line + trend** — `~/.claude/flywheel/sprint-metrics.jsonl`
   (this sprint's row + the trailing trend). Use `/hs-sw-flywheel-metrics`.
4. **Wave summaries** — the lifecycle bead's comments (stub-scan results, review
   flywheel counts per lens).

## Process

### Step 1 — Resolve the sprint

- `feature-dir` (or `--sprint`) identifies the sprint. Read `sprint-state.md` /
  `sprint-plan.md` for the bead ID list and the lifecycle bead ID.
- If none given, infer from the most recent metrics-log row + the matching beads.

### Step 2 — Categorize every failure

Tag each `[QA-FAIL]` comment and each `caught:*` fix-bead into a failure category.
Use these (extend as needed):

| Category | Signature |
|---|---|
| **reuse-miss** (rebuilt existing) | "reinvented", "already exists", duplicate of existing code |
| **stub-slippage** | stub/placeholder/`pass`/`return None` reached completion |
| **test-fraud** | skipped/xfail/`assert True`/weak assertion |
| **mock-overreach** | mocked an internal function instead of a real boundary |
| **scope-creep** | touched files outside the bead's `## Files` set |
| **contract-drift** | producer/consumer shapes diverged |
| **oversized-bead** | bead spanned >1 layer / too many ACs — botched as one unit |
| **missing-error-handling** | unhandled boundary (API/DB/IO) |
| **integration-seam** | bug where two workers' changes meet |

Record, for each: category, catch-stage (qa-fail / review / manual / pr), bead, one-line.

### Step 3 — Extract patterns (not incidents)

- A category with **2+ occurrences** this sprint is a **pattern**, not an incident.
- Note **catch-stage distribution**: if defects cluster at `caught:pr` (late/expensive),
  the wave gate is leaking — QA/review isn't catching enough early. If they cluster at
  `caught:review` (caught at the gate), the gate is working; the source (planning) is weak.
- Cross-reference the escape rate: if it's **>20%**, the dominant pattern is the thing
  blocking you from scaling — fix it first.

### Step 4 — Propose concrete corrections (human-reviewed)

For each pattern, propose a *specific, mechanical* change — map the pattern to where it
gets prevented:

| Pattern | Proposed correction |
|---|---|
| reuse-miss | strengthen the `## Steps` Search evidence / brownfield mandate; add reuse pointers in `beads-create` |
| scope-creep | tighten `## Files` declared sets; reinforce QA scope-creep check |
| contract-drift | enforce the `## Contract` consistency check in `beads-review` |
| oversized-bead | lower the right-sizing thresholds; flag in `beads-review` |
| mock-overreach | add the no-mock AC to the test bead; QA mock scan |
| recurring domain bug (e.g. null checks in API routes) | **add a project-specific line to AGENTS.md / Phase 0 enrichment** so every future bead carries it |

Output proposals as a diff-ready list — do NOT auto-apply edits to AGENTS.md or the
skills. The human approves, because these change the rules for every future sprint.

### Step 5 — Write the retrospective entry

Append to `<feature-dir>/learnings.md` (create if missing; this is the persistent,
per-project learning log):
```markdown
## Retrospective — <sprint id> (<date>)
- Escape rate: N% (trend: ↑/↓ vs prior) — <under/over 20% gate>
- Tickets: X · first-pass QA fails: Y · escape defects: review A / manual B / pr C
- Patterns (2+): <category × count>, caught mostly at <stage>
- Top driver of escape rate: <pattern>
- Proposed corrections (for review): <list, with target file>
```

### Step 6 — Report

```
Retrospective — <sprint id>
  Escape rate 31% (↑ from 18%) — OVER the 20% gate. Do not scale next sprint.
  Patterns:
    • reuse-miss ×3  (2 caught:review, 1 caught:pr) — biggest driver
    • scope-creep ×2 (caught:manual)
  Proposed corrections (your approval needed):
    1. AGENTS.md: add "this codebase has existing X helpers — grep before building"
    2. beads-create: require reuse pointer for any bead in src/services/
    3. Lower right-sizing file threshold 5 → 4 for backend beads
  Written: docs/features/<x>/learnings.md
```

## Rules

- **Propose, never auto-apply** changes to AGENTS.md / Phase 0 / skills — these are
  rule changes for every future sprint; the human gates them.
- A category needs **2+ occurrences** to be a pattern. One incident is noise.
- Weight by catch-stage: a `caught:pr` defect (escaped everything) is worth more
  attention than a `caught:review` one (the gate did its job).
- Tie the dominant pattern to the escape rate — that's the thing to fix before scaling.
- Idempotent: re-running appends a dated entry; it never rewrites prior learnings.
