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

### Step 5 — Record the retro + route the rules (record vs actuators)

A retrospective produces TWO kinds of output that belong in DIFFERENT homes. Do not
dump both into one doc — a per-feature `learnings.md` is a dead-drop nothing reads at
task time.

**5a — The RECORD (what happened):** write a dated file to
`docs/sprint-retrospectives/<YYYY-MM-DD>-<gh-or-slug>.md` with frontmatter. The
`rules_emitted` block makes the record auditable — it links each correction to where
it actually landed (CM bullet id / AGENTS.md section), so a recurring pattern is
visible across retros.
```markdown
---
date: <YYYY-MM-DD>
sprint: <sprint id>
branch: <branch>
gh_issue: <n>
beads_epic: <id>
tickets_solved: <X>
escape_rate: <0.NN>        # greppable trend across the folder
post_gate_user_found: <N>  # issues the human hit in the first hour of real use (target ≤2)
scorecard: {escape_rate: "<grade>", catch_at_layer: "<grade>", first_pass: "<grade>", velocity: "<grade>", planning_seams: "<grade>", done_calibration: "<grade>", env_hygiene: "<grade>", retro_loop: "<grade>", overall: "<grade>"}
status: <merged PR #n | open>
patterns: [<category>, ...]
rules_emitted:
  cm: [<bullet-id>, ...]
  agents_md: [<section>, ...]
  scripts: [<file/guard>, ...]
---
# Retrospective — <sprint id>
<narrative: outcome, composition note, patterns (2+), top driver, corrections table>

## Scorecard (compare each sprint — same dimensions, same strictness)
<table grading the FIXED dimensions: escape rate · catch-at-layer · first-pass
quality · velocity · planning/seams · "done" calibration (post-gate user-found
count) · env/infra hygiene · retro loop closure · overall. END with explicit
next-sprint targets.>
```

**Scorecard discipline:** BEFORE grading, read the PREVIOUS record in
`docs/sprint-retrospectives/` and put its grades side-by-side — every dimension
gets graded every sprint, same strictness, so the trend is real (a dimension
nobody re-grades silently becomes an A). The frontmatter `scorecard:` map plus
`escape_rate:` and `post_gate_user_found:` make the whole trend greppable:
`grep -h "escape_rate:\|post_gate_user_found:\|overall" docs/sprint-retrospectives/*.md`.

**5b — The ACTUATORS (what changes — so future agents act on it):**
- **Generalizable agent-behavior rules** → CM: `cm add "<rule>" --category <cat> --json`
  and capture the returned bullet id into `rules_emitted.cm`. (CM surfaces these in
  `cm context` at task time; this is the project's chosen procedural memory.)
- **Project-specific durable rules** → propose AGENTS.md diffs (per the "propose, never
  auto-apply" rule below); on approval, record the section names in `rules_emitted.agents_md`.
- **Tooling guards** (e.g. ci-check aborts on remote config) → propose the script change;
  record in `rules_emitted.scripts`.
- **Skill/Phase-0 corrections** → these change the flywheel skills repo; note them and
  the PR that carries them.

Never write the per-feature `<feature-dir>/learnings.md` — it is not consumed.

### Step 6 — Report

```
Retrospective — <sprint id>
  Escape rate 31% (↑ from 18%) — OVER the 20% gate. Do not scale next sprint.
  Patterns:
    • reuse-miss ×3  (2 caught:review, 1 caught:pr) — biggest driver
    • scope-creep ×2 (caught:manual)
  Routed:
    • CM: <bullet-id> reuse-before-build; <bullet-id> scope-discipline
    • AGENTS.md (proposed): "existing X helpers — grep before building"
    • flywheel: beads-create reuse-pointer requirement (PR #n)
  Record: docs/sprint-retrospectives/<date>-<slug>.md
```

## Rules

- **Record vs actuators (Step 5):** the dated `docs/sprint-retrospectives/` file is the
  RECORD; CM + AGENTS.md are the ACTUATORS agents read at task time. Generalizable rules
  MUST go to CM (`cm add`), not only the record — otherwise the retro is prose nobody acts on.
- **Propose, never auto-apply** changes to AGENTS.md / Phase 0 / skills — these are
  rule changes for every future sprint; the human gates them. (CM adds are fine to apply —
  they're reversible via `cm forget` and don't change shared rule files.)
- A category needs **2+ occurrences** to be a pattern. One incident is noise.
- Weight by catch-stage: a `caught:pr` defect (escaped everything) is worth more
  attention than a `caught:review` one (the gate did its job).
- Tie the dominant pattern to the escape rate — that's the thing to fix before scaling.
- Idempotent: re-running writes/updates the dated record for that sprint and never
  rewrites prior retros; CM is deduped by content; AGENTS.md edits stay human-gated.
