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
       acceptance criteria. Workers must provide evidence per step; QA
       rejects completion reports with missing step evidence.
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
   - Appropriate type (epic/feature/task/bug) and priority (0-4)
   - Record the returned bead ID for dependency wiring
4. **Phase 1b — Create TDD test beads** (see TDD section below):
   For each implementation bead with testable acceptance criteria, create a
   companion test bead.
5. **Phase 2 — Wire dependencies** (must be sequential, after all IDs exist):
   Overlay dependency structure with `bd dep add`.
   Wire TDD pairs: test bead BLOCKS impl bead (`bd dep add <impl-id> <test-id>`).
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
- Include the WHY, not just the WHAT.
- Dependencies must be correct — nothing should be unblocked that has real prereqs,
  and nothing should be blocked unnecessarily.
- Every impl bead with testable criteria must have a companion test bead.
- Test beads must block their corresponding impl beads.
- Use extended thinking for decomposition.
