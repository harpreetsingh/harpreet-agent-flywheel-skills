# The Development Flywheel

How features go from idea to production. This is the full development lifecycle
used by JoyStream — readable without tooling, executable with
[claude-workflow-skills](https://github.com/harpreetsingh/claude-workflow-skills).

Run `/flywheel` for interactive phase zoom. Run `/flywheel <phase>` to deep-dive
into any phase (shaping, planning, decomposition, sprint-planning, sprint,
wave-gate, sprint-close, review, recovery).

---

## The Compact Flywheel

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│                                                                         │
│  SHAPE ──→ PLAN ──→ REVIEW ──→ DECOMPOSE ──→ SPRINT PLAN               │
│  /shape    /plan-draft  /plan-review  /beads-create  /sprint-exec-plan  │
│                         ×4-5          /beads-review                     │
│                                                                         │
│  ──→ EXECUTE ──────────────────────────────────────────→ CLOSE          │
│      /sprint-go                                          /land-the-plane│
│      ┌──────────────────────────────────────────────┐                   │
│      │ Director orchestrates:                        │                   │
│      │  Phase 0 → Wave N [ TDD → QA → Gate → Review │                   │
│      │                      Flywheel ] → Sprint Close│                   │
│      │  Sprint Close: docs-int, docs-ext,            │                   │
│      │    fresh-eyes, land-the-plane                 │                   │
│      └──────────────────────────────────────────────┘                   │
│                                                                         │
│  ──→ HUMAN REVIEW ──→ MERGE ──→ (next feature loops back to SHAPE)      │
│      PR review, manual QA, bd close                                     │
│                                                                         │
│  Sprint failed? ──→ /sprint-recover ──→ re-enter at SPRINT PLAN         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The Full Flywheel

```
╔══════════════════════════════════════════════════════════════════════════╗
║                         THE FLYWHEEL                                    ║
╚══════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────┐
│ 1. SHAPING                                          Human + Claude     │
│                                                                         │
│    /shape #42                                                           │
│         ↓                                                               │
│    5-round interview: Problem → Appetite → Solution → Rabbit Holes      │
│         → No-Gos                                                        │
│         ↓                                                               │
│    OUTPUT: docs/features/<slug>/pitch.md                                │
│            docs/features/<slug>/planning-context/                       │
│                                                                         │
│    ✓ GitHub Issue updated with pitch link                               │
│    ✓ Label: ready-to-bet                                                │
└────────────────────────────────────────┬────────────────────────────────┘
                                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 2. PLANNING                                         Human + Claude     │
│                                                                         │
│    /plan-draft docs/features/<slug>/                                    │
│         ↓                                                               │
│    Reads pitch.md + planning-context/ → synthesizes PLAN.md             │
│    (architecture, deliverables, CLI commands, diagrams)                  │
│         ↓                                                               │
│    /repeat /plan-review docs/features/<slug>/PLAN.md                    │
│         ↓                                                               │
│    Each round: severity-rated proposals → user approves → apply         │
│    Converges after 4-5 rounds (auto-detected by /repeat)                │
│         ↓                                                               │
│    OUTPUT: docs/features/<slug>/PLAN.md (refined)                       │
└────────────────────────────────────────┬────────────────────────────────┘
                                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 3. DECOMPOSITION                                    Human + Claude     │
│                                                                         │
│    /beads-create docs/features/<slug>/PLAN.md                           │
│         ↓                                                               │
│    PLAN → epics → tasks + TDD test beads                                │
│    Wires dependencies (test beads block impl beads)                     │
│    CLI beads for every API/UI feature                                   │
│         ↓                                                               │
│    /test-coverage <project-dir>                                         │
│         ↓                                                               │
│    Verify: full unit tests (no mocks), e2e integration tests with       │
│    detailed logging, CLI tests. Creates beads for every gap.            │
│         ↓                                                               │
│    /beads-review                                                        │
│         ↓                                                               │
│    Structural review: orphans, cycles, TDD gaps, domain balance         │
│         ↓                                                               │
│    OUTPUT: beads epic with all tickets wired                            │
└────────────────────────────────────────┬────────────────────────────────┘
                                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 4. SPRINT PLANNING                                  Human + Claude     │
│                                                                         │
│    /sprint-exec-plan                                                    │
│         ↓                                                               │
│    Inventory → TDD pairing → wave analysis → model tiers (opus/sonnet)  │
│    → domain balance → team topology → ASCII diagram                     │
│         ↓                                                               │
│    OUTPUT: tmp/sprint-exec-plan.md                                      │
│            docs/features/<slug>/sprint-plan.md (persistent copy)        │
│                                                                         │
│    Optional: /sprint-go --dry-run  (preview without launching)          │
└────────────────────────────────────────┬────────────────────────────────┘
                                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 5. SPRINT EXECUTION                                 Autonomous         │
│                                                                         │
│    /sprint-go                                                           │
│         ↓                                                               │
│    Spawns Director (background) → Director creates team                 │
│    Director spawns: workers (≤5) + QA agent(s) (1-2)                    │
│         ↓                                                               │
│  ┌─── Phase 0: Ticket Sufficiency Review ───────────────────────────┐   │
│  │  Enrich every bead for self-sufficiency. Create missing TDD      │   │
│  │  pairs. No work assigned until Phase 0 completes.                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│         ↓                                                               │
│  ┌─── Wave N (repeats per wave) ────────────────────────────────────┐   │
│  │                                                                   │   │
│  │  ASSIGN: test beads → Worker A (red phase)                        │   │
│  │       ↓                                                           │   │
│  │  SELF-REVIEW: Worker re-reads own code with fresh eyes, fixes     │   │
│  │       ↓                                                           │   │
│  │  QA VERIFY: tests fail for right reasons, no impl code            │   │
│  │       ↓                                                           │   │
│  │  ASSIGN: impl beads → Worker B (green phase, DIFFERENT worker)    │   │
│  │       ↓                                                           │   │
│  │  SELF-REVIEW: Worker re-reads own code with fresh eyes, fixes     │   │
│  │       ↓                                                           │   │
│  │  QA VERIFY: tests pass, acceptance criteria met                   │   │
│  │       ↓                                                           │   │
│  │  DEEP REVIEW (rolling, mid-wave):                                 │   │
│  │    Idle workers do alternating explore/cross-agent review          │   │
│  │    UBS first → explore → cross-review → converge (2 clean rounds) │   │
│  │       ↓                                                           │   │
│  │  ╔═══ WAVE GATE (hard — blocks Wave N+1) ═══════════════════╗    │   │
│  │  ║ 0. Stub scan + test integrity scan (grep, mechanical)    ║    │   │
│  │  ║ 0c. UBS scan (security + null safety + async, mechanical) ║    │   │
│  │  ║ 1. All tickets individually QA-passed                     ║    │   │
│  │  ║ 2. Integration quality gates (ruff + pytest / lint + tsc) ║    │   │
│  │  ║ 3. Review flywheel (3-4 ephemeral agents in parallel):    ║    │   │
│  │  ║    • CORRECTNESS — bugs, logic errors, stubs, fakes      ║    │   │
│  │  ║    • SECURITY — arch-level: trust boundaries, auth flows  ║    │   │
│  │  ║      (UBS already caught pattern-level security issues)    ║    │   │
│  │  ║    • COMPACTION — reinvention, UX drift, dead code, bloat   ║    │   │
│  │  ║    • UX (frontend waves only) — patterns, a11y, states    ║    │   │
│  │  ║ 4. QA smoke test                                          ║    │   │
│  │  ║ 5. Checkpoint written, lifecycle bead updated              ║    │   │
│  │  ║ 6. Human review (Wave 1: BLOCKING / Wave 2+: async)      ║    │   │
│  │  ╚══════════════════════════════════════════════════════════╝    │   │
│  │       ↓                                                           │   │
│  └── next wave ─────────────────────────────────────────────────────┘   │
│         ↓                                                               │
│  ┌─── Sprint Close ────────────────────────────────────────────────┐    │
│  │  1. Final quality gates + UBS full project scan                  │    │
│  │  2. /test-coverage — verify full unit + e2e + CLI test coverage  │    │
│  │  3. /docs-gen-int → architecture.md, api.md, cli.md, etc.      │    │
│  │  4. /docs-gen-ext → docs/site/features/, guides/, reference/    │    │
│  │  5. /fresh-eyes <feature-dir> (code + plan + docs + beads)      │    │
│  │  6. /land-the-plane → commit + push + PR                        │    │
│  │  7. Lifecycle bead completed, final checkpoint written           │    │
│  │  8. Summary to user → shutdown                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│    OUTPUT: docs/features/<slug>/sprint-state.md (completed)             │
│            docs/features/<slug>/architecture.md, api.md, cli.md, ...    │
│            docs/site/features/<slug>/... , guides/, reference/          │
│            All beads labeled qa-passed (human closes after review)       │
└────────────────────────────────────────┬────────────────────────────────┘
                                         ↓
┌─────────────────────────────────────────────────────────────────────────┐
│ 6. HUMAN REVIEW                                     Human              │
│                                                                         │
│    PR review on GitHub                                                  │
│    Manual QA on staging                                                 │
│    bd close <id> for each verified bead                                 │
│    Merge feature branch → main → production                             │
│                                                                         │
│    Sprint failed or incomplete?                                         │
│         ↓                                                               │
│    /sprint-recover <feature-dir> — triage, fix beads, re-enter at 4.    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Artifact Map

Every feature produces a consistent set of artifacts across its lifecycle:

```
docs/features/<slug>/
├── pitch.md              ← /shape
├── planning-context/     ← /shape (evidence bag: research, competitor analysis)
├── PLAN.md               ← /plan-draft → /plan-review ×4-5
├── sprint-plan.md        ← /sprint-exec-plan (persistent copy)
├── sprint-state.md       ← Director (checkpoint, updated per wave gate)
├── architecture.md       ← /docs-gen-int (sprint close)
├── api.md                ← /docs-gen-int
├── cli.md                ← /docs-gen-int
├── data-model.md         ← /docs-gen-int
├── what-shipped.md       ← /docs-gen-int
└── lessons.md            ← /docs-gen-int

docs/site/
├── features/<slug>/      ← /docs-gen-ext (concept docs)
├── guides/               ← /docs-gen-ext (how-to guides)
├── reference/api/        ← /docs-gen-ext (API reference)
└── reference/cli/        ← /docs-gen-ext (CLI reference)

Beads (bd):
├── Epic: feature epic
├── Sprint Lifecycle bead ← Director (meta-ticket tracking sprint checklist)
├── Test beads            ← /beads-create (red phase — written first, must fail)
├── Impl beads            ← /beads-create (green phase — make tests pass)
└── Bug/review beads      ← bug-hunter (filed during review flywheel)
```

---

## Skill & Agent Reference

### Skills (human-invoked)

| Skill | What it does | Inputs | Outputs |
|-------|-------------|--------|---------|
| `/shape` | 5-round Shape Up interview: problem, appetite, solution, rabbit holes, no-gos | GitHub Issue # | `pitch.md`, `planning-context/` |
| `/plan-draft` | Synthesize pitch + research into structured PLAN with diagrams and CLI spec | Feature dir or files | `PLAN.md` |
| `/plan-review` | One round of deep review with severity-rated proposals and diffs | `PLAN.md` path | Approved edits to PLAN |
| `/beads-create` | Decompose PLAN into epics, tasks, and TDD test beads with dependencies | `PLAN.md` path | Beads epic with wired tickets |
| `/beads-review` | Structural review: orphans, cycles, TDD gaps, domain balance, scope | — | Fixes applied to beads |
| `/sprint-exec-plan` | Wave analysis, model tiers, team topology, ASCII deployment diagram | Beads backlog | `sprint-exec-plan.md`, `sprint-plan.md` |
| `/sprint-go` | Launch sprint: spawn Director in background. `--dry-run` to preview. | Sprint plan | Director agent (autonomous) |
| `/docs-gen-int` | Synthesize internal engineering docs from sprint artifacts | Feature dir | `architecture.md`, `api.md`, `cli.md`, `data-model.md`, `what-shipped.md`, `lessons.md` |
| `/docs-gen-ext` | Extract user-facing docs from internal artifacts | Feature dir | `docs/site/features/`, `guides/`, `reference/` |
| `/fresh-eyes` | Cold-read audit of any artifact: code, plans, docs, research, beads | File, dir, or `--beads` | Fixes (code/docs) or report (plans/beads) |
| `/test-coverage` | Analyze test gaps, create beads for missing tests | Directory | Test beads with scenarios |
| `/ux-polish` | Deep UI/UX + CLI UX scrutiny targeting Stripe-level quality | Component or dir | Fixes or beads for gaps |
| `/land-the-plane` | Quality gates → bd preflight → logical commits → push → PR | — | Commits + PR URL |
| `/sprint-recover` | Triage failed sprint, reopen beads, create TDD pairs, repair deps | Feature dir or triage doc | Clean backlog ready for `/sprint-exec-plan` |
| `/repeat` | Run any review skill in rounds until convergence | `[N] /<skill> [args]` | Converged artifact + summary table |
| `/flywheel` | Render this lifecycle map, zoom into any phase | Phase name (optional) | `FLYWHEEL.md` + interactive display |

### Agents (spawned by Director during sprints)

| Agent | Role | Tools | Key behavior |
|-------|------|-------|-------------|
| **Director** | Sprint coordinator. Creates team, assigns work, runs wave gates, manages lifecycle bead. Never implements. | All | Opus-tier. Spawns workers + QA. Writes checkpoints to `sprint-state.md`. |
| **QA** | Independent verifier. Checks every ticket against acceptance criteria. Never implements. | Read, Grep, Glob, Bash, SendMessage | Permanent teammate. Reports PASS/FAIL. Strict — 80% of criteria = FAIL. |
| **Bug Hunter** | Ephemeral wave-gate reviewer. One lens per instance (correctness, security, compaction, UX). | Read, Grep, Glob, Bash, SendMessage | Spawned fresh per wave. Files beads for issues. Shuts down after reporting. |
| **Peer Reviewer** | Mid-wave deep reviewer. Two modes: EXPLORE (random exploration) and CROSS-REVIEW (boundary-focused). | Read, Edit, Grep, Glob, Bash | Sent to idle workers during implementation. Alternating modes converge on 2 clean rounds. |
| **Workers** | General-purpose implementation agents. Write code, tests, docs. | All | Sonnet or Opus tier. Different worker writes tests vs implementation (TDD). Cap: 5. |

---

## Key Concepts

### TDD Enforcement
Test-driven development is **structurally enforced**, not honor-system. Test beads
**block** their implementation beads via `bd dep add`. A different worker writes
the tests (red phase) than the one who writes the implementation (green phase).
QA verifies tests fail for the right reasons before implementation begins.

### Wave Gates
No Wave N+1 work begins until the gate passes: stub scan → UBS scan → individual
QA → integration quality gates → review flywheel (3-4 parallel lens agents) →
smoke test → checkpoint → human review (Wave 1: blocking, Wave 2+: async).
Mid-wave deep review (alternating explore/cross-agent) runs on idle workers during
implementation and converges before the gate.

### Sprint State & Recovery
The Director writes checkpoints to `<feature_dir>/sprint-state.md` after every
wave gate. A lifecycle meta-bead tracks the full sprint checklist. If context is
lost (compaction, crash), a new Director reads the checkpoint and resumes.

### Bead Lifecycle (during sprints)
```
open → in_progress (worker claims) → qa-passed label (QA verified) → closed (human)
```
Agents **never** run `bd close`. The human closes beads after PR review.

### Branch Strategy
```
feature/<name> or fix/<name> → PR → main → production
```
No `development` branch. No direct pushes to `main` or `production`.

---

Generated by `/flywheel` on 2026-04-21. Run `/flywheel` for interactive phase zoom.
