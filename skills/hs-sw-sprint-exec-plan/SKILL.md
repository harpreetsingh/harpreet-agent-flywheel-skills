---
name: hs-sw-sprint-exec-plan
description: Analyze beads into waves, label cost tiers, design team topology, generate ASCII diagram
---

# /sprint-exec-plan — Sprint Execution Plan

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → ★SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Analyze beads into waves + team. Last step before go.   │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Read-only analysis of your beads backlog. Produces a sprint brief with wave plan,
cost tiers, team topology, and ASCII deployment diagram. No team creation — this
is the thinking step before `/hs-sw-sprint-go`.

## Process

### Step 1 — Inventory

- `bd list --status=open` to get all open tickets
- `bd show <id>` for each ticket to get full context
- `bd graph --all` for dependency structure
- Read AGENTS.md for project context, quality gates, verification entry points

### Step 2 — Verification Entry Points

- Check AGENTS.md for a "Verification Entry Points" section
- If configured: create missing verification tickets with `bd create`
- Wire as dependents of implementation tickets they verify
- Place in final wave
- If no section: note and suggest adding one

### Step 3 — TDD Pairing Check

Before wave analysis, verify TDD structure:

- For each impl bead with testable acceptance criteria: does a companion test bead exist?
- Does the test bead block the impl bead?
- If missing: create the test bead with `bd create` and wire dependencies
- Test beads should be in the same wave or one wave before their impl bead
- Log any test beads created: `bd comments add <id> "Sprint plan: created TDD pair"`

### Step 4 — Wave Analysis

- Build DAG from beads dependencies (including TDD pairs)
- Group into waves: Wave 1 = no deps, Wave N = deps all in 1..N-1
- Name waves by dominant work type (Foundation / Core / Integration / Polish)
- Verify test beads appear in same or earlier wave than their impl beads
- Flag cycles — must resolve before proceeding

### Step 5 — Model Tier Labeling

Apply tiers via `bd update <id> --add-label tier:<tier>`:

- `opus`: architectural decisions, complex multi-file refactors, high fan-out (blocks 3+), protocol design, anything moderately complex or above
- `sonnet`: standard features, API endpoints, UI components, config, boilerplate, test writing, docs, mechanical tasks

Do NOT use `haiku` tier. All sprint work uses either opus or sonnet.

Respect existing ticket labels — don't overwrite user-assigned ones.

### Step 6 — Domain Balance Check

Count beads by domain (backend, frontend, infra, tests). Flag imbalances:

- **BLOCK** if any domain has impl beads but no worker assigned to it
- **WARN** if >30% of beads are frontend but no frontend-specialized worker
- **WARN** if frontend and backend beads exist in the same wave but only one
  domain has a worker
- **WARN** if a domain has impl beads but zero test beads

Present domain distribution to user:
```
Backend: 15 impl + 8 test = 23 beads
Frontend: 10 impl + 5 test = 15 beads
Infra: 3 impl + 0 test = 3 beads  ⚠️ no test coverage
```

### Step 7 — Team Topology

- Agent count: total tickets / 4, capped at 5 workers
- Logical groupings: analyze domains (backend/frontend/infra or by label)
- **Every domain with beads MUST have at least one worker** — this is the rule
  that prevents "backend done, frontend skipped"
- Manager layer: only if 4+ workers; otherwise Director manages directly
- Director: always one, always opus-tier
- QA agent: always one, always present (does not count toward worker cap)
- TDD consideration: plan for test-writer / implementer separation on opus tickets

### Step 8 — Generate Sprint Brief

Two artifacts:

**A. ASCII Deployment Diagram:**
```
Wave 1 (Foundation)     Wave 2 (Core)        Wave 3 (Polish)
┌─────────────────┐    ┌────────────────┐    ┌────────────────┐
│ T-01 schema  ◆  │    │ T-04 API    ●  │    │ T-07 tests  ●  │
│ T-02 models  ◆  │───▶│ T-05 UI     ●  │───▶│ T-08 docs   ●  │
│ T-03 config  ●  │    │ T-06 hooks  ●  │    │ T-09 demo   ●  │
└─────────────────┘    └────────────────┘    └────────────────┘
◆ = opus  ● = sonnet

Director (Opus) — coordinator only, never implements
├── QA Agent — independent verification, never implements
├── Backend Worker-1: T-01 (test), T-04 (impl)
├── Backend Worker-2: T-02 (test), T-05 (impl)
└── Frontend Worker-3: T-06 (test), T-07 (impl), T-08, T-09
```

**B. Director Brief** — self-contained markdown with:
- Project, branch, tech stack, quality gates
- Wave plan with ticket IDs, titles, tiers, dependencies
- Team topology with agent assignments
- Role switching rules (impl → test → docs → marketing → fresh-eyes)
- Autonomy mandate

### Step 9 — Persist + Present

- Determine the feature directory. Look at the beads epic or ask the user:
  "What feature directory should I use? (e.g., `docs/projects/features/org-management/`)"
- Write full plan to `tmp/sprint-exec-plan.md` (for sprint-go to read)
- **Also write to `<feature_dir>/sprint-plan.md`** — this is the persistent copy
  that lives alongside PLAN.md and pitch.md. Include the `feature_dir` path in
  the plan so the Director knows where to write checkpoints.
- **Generate `tmp/sprint-status.sh`** — sprint status line script for Claude Code:
  - Get the project root absolute path via `pwd`
  - Hardcode all ticket IDs and wave-to-ticket mapping (known from Step 4)
  - Script queries `bd show <all-ids> --json` in one call, formats output as:
    `Sprint: ■■▣□□ 2/5 done · W1:✓ W2:◐ W3:○`
  - **Completion = `qa-passed` label** (agents never close beads — humans do after review)
  - Use symbols: `■` qa-passed (done), `▣` in_progress, `□` open; `✓` wave done, `◐` wave active, `○` wave pending
  - Detection logic: `if "qa-passed" in item.get("labels", []) → "done"`, else use `item["status"]`
  - Falls back to `"Sprint: loading..."` if bd or python fails
  - See `docs/features/sprint-status-line/statusupdate.md` for full script template (copy it exactly)
  - Make executable: `chmod +x tmp/sprint-status.sh`
  - Verify it runs: `bash tmp/sprint-status.sh`
- **Configure the project-level `.claude/settings.json`** with the status line:
  - This is `<project-root>/.claude/settings.json` — NOT `~/.claude/settings.json` (never write to global settings)
  - Read existing `.claude/settings.json` (or start from `{}` if missing)
  - Merge in: `"statusLine": {"type": "command", "command": "bash <abs-path>/tmp/sprint-status.sh"}`
  - Write back — preserve all other existing settings
- Present ASCII diagram + topology + cost summary in conversation
- Tell user: "Review and adjust, or run `/hs-sw-sprint-go` to launch"

## Rules

- Use extended thinking for wave analysis and topology decisions
- All ticket operations use `bd` CLI
- If dependency graph has cycles: report and stop
- Respect existing ticket labels — don't overwrite user-assigned ones
- This is read-only analysis (except verification ticket creation and tier labels)
