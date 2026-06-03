---
name: hs-sw-beads-create
description: Create comprehensive beads (epics/tasks/subtasks) with dependencies from a plan
argument-hint: [plan-file-path]
---

# /beads-create — Plan to Beads

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → ★DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Break PLAN into executable beads with TDD pairs.        │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Read the plan at `$ARGUMENTS` (default: `PLAN.md`) and create a comprehensive,
granular set of beads with full dependency structure.

## Process

1. Read the plan file thoroughly
2. Decompose into epics → tasks → subtasks. Maintain traceability — note which
   plan section each bead maps to in the description.
3. **Phase 1 — Create beads** (can be parallelized):
   For each bead, create with `bd create`:
   - Clear imperative title
   - Detailed description including:
     - Which plan section this implements and why
     - Background and reasoning/justification
     - Acceptance criteria — **mechanically verifiable** conditions that a QA
       agent can check with grep, curl, test output, or file reads. Each
       criterion must specify a CONCRETE observable:
       - BAD: "user can search" / "it works" / "handles errors"
       - GOOD: "GET /api/search?q=test returns 200 with `results` array"
       - GOOD: "SearchBar component renders input with placeholder 'Search...'"
       - GOOD: "`bd search test` returns matching beads in table format"
       - GOOD: "pytest test_search.py passes with 5+ assertions"
       If a criterion can't be verified by reading code or running a command,
       rewrite it until it can. Vague criteria produce vague tests which pass
       with stubs.
     - CLI commands this bead must deliver (if applicable) — command name,
       flags, `--json` output format. A bead that delivers an API endpoint
       or UI feature MUST also specify its CLI counterpart.
     - Relevant considerations and gotchas
     - How it serves the overarching project goals
     - **Standard execution steps** — every bead gets a `## Steps` section
       in its description. This is the execution protocol — not more
       acceptance criteria. It is a **verification contract**: every step you
       write here WILL be checked by QA for execution evidence, 1:1. The worker
       must provide evidence per step, and QA fails any step that lacks it. So
       write only steps you expect to be verified — and if a bead needs a step
       beyond the standard four, add it here and it will be enforced like the
       rest. A bead with no `## Steps` section is incomplete.
       ```
       ## Steps
       - [ ] **Search** — Three layers, in order:
             1. `cm context "<bead title>" --json` — prior rules, anti-patterns,
                past solutions from CM playbook
             2. `cass search "<concept>" --json --limit 5` — find past sessions
                that solved similar problems
             3. Serena (`find_symbol`, `search_symbols`, `find_references`) +
                grep/glob fallback — find existing code in the current codebase
             Evidence: CM rules found, CASS sessions found, Serena/grep results.
       - [ ] **Read** — read all identified files in full. Evidence: key
             insights that shaped the implementation approach.
       - [ ] **Implement** — make the change. Evidence: files changed and
             approach chosen.
       - [ ] **Verify** — run tests and linters. Evidence: test output
             with pass/fail counts.
       ```
       These 4 steps map to `[[steps]]` in a future GasCity formula. When
       migrating to GasCity, `gc sling` enforces step order structurally —
       the checklist becomes redundant but the step definitions carry over.
     - **Shared interface contract** — include ONLY if this bead produces or
       consumes an interface another bead depends on: an API route, a type/model,
       an event, or a DB schema. Add a `## Contract` block defining the EXACT
       shape and copy it **byte-identical into both the producing bead and every
       consuming bead**, naming the producer as source of truth:
       ```
       ## Contract: invites API  (source of truth: <producer bead id>)
       POST /api/v1/invites
       Request:  {email: str, role: MemberRole(admin|member|viewer), workspace_id: UUID}
       Response 201: {id: UUID, status: 'pending'}
       ```
       Why: at 10+ parallel agents the producer and consumer are built by
       different agents simultaneously. The self-sufficiency rule makes each bead
       inline its own shapes — but if each *invents* the shape independently, they
       diverge silently and integration breaks. One identical block, copied to
       both, is the anti-drift anchor. (Right-sizing splits every cross-layer
       feature into exactly these producer/consumer pairs — so most multi-bead
       features need a contract.)
   - Appropriate type (epic/feature/task/bug) and priority (0-4)
   - Record the returned bead ID for dependency wiring
4. **Phase 1b — Create TDD test beads** (see TDD section below):
   For each implementation bead with testable acceptance criteria, create a
   companion test bead.
5. **Phase 2 — Wire dependencies** (must be sequential, after all IDs exist):
   Overlay dependency structure with `bd dep add`.
   Wire TDD pairs: test bead BLOCKS impl bead (`bd dep add <impl-id> <test-id>`).
   **For every producer→consumer edge that crosses an interface** (API, type,
   event, schema): confirm the `## Contract` block is present and byte-identical
   in both beads now that the producer ID exists to name as source of truth.
   `bd update` the consumer if the producer's shape was finalized after creation.
6. **Phase 3 — Verify**:
   - `bd graph --all` to check the structure visually
   - `bd ready` to confirm which beads are immediately actionable
   - Flag anything that looks wrong (orphaned beads, everything blocked, etc.)
   - Verify every impl bead with testable criteria has a companion test bead

## TDD: Test-First Beads

Every implementation bead with testable acceptance criteria MUST have a companion
test bead. This enforces red-green-refactor at the planning level — tests are
written and verified FAILING before implementation begins.

### When to create a test bead

Create a companion test bead when the impl bead:
- Has an API endpoint (test request/response contracts)
- Has business logic (test inputs/outputs/edge cases)
- Has a UI component (test route exists, component renders, build passes)
- Has a data model (test schema, constraints, RLS policies)

Do NOT create test beads for:
- Config/boilerplate (no logic to test)
- Documentation tasks
- Epic-level tracking beads

### Test bead format

```
Title: "Write tests for <feature> (red)"
Description:
  - What to test (derived from impl bead's acceptance criteria)
  - Expected test file paths
  - Key assertions that must exist
  - "Tests MUST FAIL when written — there is no implementation yet"
Acceptance criteria:
  - [ ] Test file(s) created at expected path(s)
  - [ ] Tests run and FAIL (red phase) — not import errors, real assertion failures
  - [ ] N+ assertions covering the impl bead's acceptance criteria
  - [ ] No implementation code written (test-only changes)
Type: task
```

### Dependency wiring for TDD

```
bd dep add <impl-bead-id> <test-bead-id>
```

The test bead BLOCKS the impl bead. Implementation cannot start until tests
exist and are verified failing. This is structural enforcement — not honor-system.

### Wave placement

Test beads go in the same wave as or one wave before their impl bead. The
dependency graph naturally prevents impl from starting before tests exist.

## Domain Coverage Check

After all beads are created, verify domain balance:

- Count beads by domain: backend, frontend, infrastructure, tests
- If any domain has >30% of total beads, flag it
- If frontend and backend beads exist but test beads only cover one domain, flag it
- If a domain has impl beads but zero test beads, STOP and create them

## Rules

- Every bead must be totally self-contained and self-documenting — a future
  agent picking up any bead should have full context without reading anything else.
- **Right-size every bead — one bead, one layer, one deliverable.** Oversized
  beads are the #1 cause of rework. Hard limits:
  - A bead must NOT span more than one architectural layer. If a feature needs
    DB/schema + API + UI, create a SEPARATE bead per layer and wire a dependency
    between them (the producing layer's bead defines the interface; the consuming
    layer's bead references it). Never one "build the whole feature" bead.
  - ≤5 acceptance criteria per bead. More than that = split by deliverable.
  - ≤~5 files in a bead's expected scope. Broader = split.
  - One deliverable per title. A title with "X and Y" is two beads.
  - Don't go too granular either: a <30-min trivial change with no test pairing
    belongs merged into its sibling, not as its own bead.
  `/hs-sw-beads-review` enforces these mechanically — create them right the first time.
- Include the WHY, not just the WHAT.
- Dependencies must be correct — nothing should be unblocked that has real prereqs,
  and nothing should be blocked unnecessarily.
- Every impl bead with testable criteria must have a companion test bead.
- Test beads must block their corresponding impl beads.
- Use extended thinking for decomposition.
