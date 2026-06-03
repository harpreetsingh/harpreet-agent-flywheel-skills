---
name: hs-sw-sprint-go
description: Launch multi-agent sprint from execution plan — spawn director who owns the team
argument-hint: [--dry-run]
---

# /sprint-go — Launch Sprint

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → ★EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Launch. Spawns Director, walk away. --dry-run to preview│
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Reads the persisted sprint plan from `/hs-sw-sprint-exec-plan` and launches
a multi-agent sprint. This is the walk-away moment.

**Key architecture:** Director creates the team and spawns workers — making Director
the system-level team-lead who receives all worker messages directly. The user's
session exits the loop after spawning Director.

## Dry-Run Mode

If `$ARGUMENTS` contains `--dry-run`: simulate the entire sprint without spawning
any agents, creating any teams, or modifying any beads. This is a preview of what
the Director would do.

### Dry-Run Process

1. Read `tmp/sprint-exec-plan.md` (same as live mode)
2. Read all beads referenced in the plan (`bd show <id>` for each)
3. **Simulate the full Director lifecycle** and output a narrative:

```markdown
## Sprint Dry-Run: <feature-name>

### Team
- Director (opus) — coordinator
- Worker-1 (backend): T-01, T-04, T-07
- Worker-2 (frontend): T-02, T-05, T-08
- QA-1: all tickets

### Phase 0 — Ticket Sufficiency
[For each bead: PASS or NEEDS ENRICHMENT with what's missing]

### Wave 1 — Foundation (estimated: X tickets)
1. **Assign:** T-01-test → Worker-1 (red phase), T-02-test → Worker-2 (red phase)
2. **QA verifies** test beads (tests fail for right reasons, no impl code)
3. **Assign:** T-01 → Worker-2 (green phase — different worker), T-02 → Worker-1
4. **QA verifies** impl beads (tests pass, acceptance criteria met)
5. **Wave Gate:**
   - Integration quality gates: `uv run ruff check . && uv run pytest -v`
   - Review flywheel: correctness + security + compaction (3 agents)
   - Smoke test: health endpoint, new endpoints from this wave
6. **Checkpoint written** to <feature_dir>/sprint-state.md

### Wave 2 — Core (estimated: X tickets)
[Same structure...]

### Sprint Close
1. Quality gates (full suite)
2. docs-gen-int → writes architecture.md, api.md, cli.md, etc. to <feature_dir>/
3. docs-gen-ext → writes to docs/areas/site/features/<name>/, guides/, reference/
4. fresh-eyes → cold-read audit of entire <feature_dir> (code + plan + docs + beads)
5. land-the-plane → commit + push
6. Lifecycle bead marked complete, final checkpoint written

### Summary
- Total agents: N workers + N QA + ~N×3 review agents (ephemeral)
- Total waves: N
- Estimated tickets: N (N impl + N test)
- Artifacts produced: sprint-plan.md, sprint-state.md, architecture.md, api.md,
  cli.md, data-model.md, what-shipped.md, lessons.md, docs/areas/site/...
```

4. **Flag risks:** highlight any beads that look problematic (missing acceptance
   criteria, vague scope, no TDD pair, suspicious dependencies)
5. Ask: "This is what the sprint will do. Proceed with `/sprint-go` (no --dry-run) to launch, or adjust?"

**Dry-run does NOT:**
- Create teams, spawn agents, or modify beads
- Write any files (no checkpoint, no sprint-plan.md)
- Run quality gates or tests

---

## Live Mode (default — no --dry-run flag)

### Step 1 — Read Sprint Plan

- Read `tmp/sprint-exec-plan.md`
- If missing: tell user to run `/hs-sw-sprint-exec-plan` first and stop
- Present quick summary: "About to launch N workers across K waves. Confirm?"

### Step 1b — Status Line Check

- Check `.claude/settings.json` for `statusLine` config
- If missing or `tmp/sprint-status.sh` doesn't exist: warn the user
  ```
  ⚠ Sprint status bar not configured. Run /hs-sw-sprint-exec-plan to generate it,
    or you won't see live progress in the Claude Code status bar.
  ```
- If present: confirm it's active — `bash tmp/sprint-status.sh` and show the output

### Step 2 — Approval Gate

- Show: ticket count, agent count, estimated cost tier distribution
- Show: domain balance (backend/frontend/infra coverage)
- Show: TDD coverage (how many impl beads have companion test beads)
- **Wait for explicit "go" from user**
- Options: go / cancel

### Step 3 — Spawn Director

- `Agent(subagent_type="hs-sw-sprint-director", name="director", run_in_background=true)`
- Pass the FULL sprint brief as the prompt, including:
  - Complete `tmp/sprint-exec-plan.md` content
  - **`feature_dir`** path (e.g., `docs/projects/features/org-management/`) — the Director
    writes sprint-state.md checkpoints here and reads sprint-plan.md for recovery
  - Team topology (worker names, types, model tiers, ticket assignments)
  - Team name to create: `sprint-<date>-<project>`
  - Quality gates and project context from AGENTS.md/CLAUDE.md
- Director will create the team, spawn workers, and manage autonomously

### Step 4 — Exit

- Report to user: "Director launched in background. It will create the team, spawn N workers + QA agent(s)."
- "You can walk away. Director manages everything. Return to check verification entry points."
- "Beads will have the `qa-passed` label when sprint finishes — you close them after review with `bd close`."
- "When the sprint is done, run `/sprint-close` to close all qa-passed beads and remove the status bar. Use `--dry-run` first to preview. If the bar shows stale data before then, run `/sprint-status-sync`."
- **Do NOT create a team. Do NOT spawn workers. Do NOT stay in the loop.**

## Rules

- Never proceed without explicit user approval (live mode only — dry-run needs no approval)
- Director creates the team (NOT the launcher) — this is critical for message routing
- Director gets the full sprint brief + team topology in its spawn prompt
- Director spawns QA agent(s) alongside workers — QA verifies all completions
- 1-3 workers → 1 QA agent; 4-5 workers → 2 QA agents (split by domain)
- Cap at 5 concurrent workers (enforced by Director); QA agents do not count toward cap
- Wave gates are hard — no Wave N+1 work until gate passes (QA + quality gates + review flywheel + smoke test)
- Review flywheel: 3 parallel lenses (correctness, security, compaction) + UX for frontend waves
- Review agents are ephemeral — spawned fresh per wave, shut down after reporting
- Sprint close includes docs generation: `/hs-sw-docs-gen-int` (internal) + `/hs-sw-docs-gen-ext` (external to `docs/areas/site/`)
- Workers are general-purpose (full tool access)
- NO Tasks — beads (`bd`) is the only tracking system
- Launcher exits immediately after spawning Director
- Requires Claude Code team features
