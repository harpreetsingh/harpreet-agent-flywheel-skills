---
name: hs-sw-test-coverage
description: Analyze test coverage gaps and create beads for missing tests — no mocks, real e2e with detailed logging
argument-hint: [directory]
---

# /test-coverage — Test Gap Analysis

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE   │
│ ★ YOU ARE HERE: Three touchpoints —                                     │
│   1. Planning (verify test strategy in PLAN.md)                         │
│   2. Decompose (after /beads-create — find missing test beads)          │
│   3. Sprint close (verify actual coverage matches plan)                 │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Analyze the codebase (or `$ARGUMENTS` directory) for test coverage gaps and
create beads for every missing test.

## Discovery

Before creating beads, understand the project's test infrastructure:
1. Detect the test framework(s) in use (pytest, jest, vitest, etc.)
2. Find the test directory conventions (tests/, __tests__/, *.test.*, *.spec.*)
3. Map source files to their corresponding test files

## What to evaluate

### 1. Unit tests — no mocks, no fakes

Does every module/function have real unit tests?

**The no-mock mandate:** Do NOT use mocks or fakes unless you are at a true
system boundary. Mocks hide bugs — a mocked test passes even when the real
integration is broken.

| Acceptable (system boundary)         | NOT acceptable (internal)               |
|--------------------------------------|-----------------------------------------|
| HTTP client (use httpx test client)  | Mocking your own service functions      |
| Database driver (use test DB)        | Mocking your own repository layer       |
| External API (use recorded fixtures) | Mocking internal class methods          |
| File system (use tmp dirs)           | Mocking function return values          |
| Time/clock (freeze time)            | `mock.patch` on anything you wrote      |

If a test requires more than 2 lines of mock setup, it's testing mocks, not code.
Rewrite it to use a real test database, real HTTP test client, or real filesystem.

### 2. E2E integration tests — with detailed logging

Are there complete end-to-end integration test scripts?

**What "detailed logging" means:**
- Every test step logs what it's doing: `LOG: Creating user with email=test@x.com`
- Every assertion logs expected vs actual: `ASSERT: status=201, body.id exists`
- Every failure includes full context: request payload, response body, headers, timing
- Test output should be diagnosable without re-running: a developer reading the
  log should understand exactly what happened and where it broke

**What to cover:**
- Critical user workflows end-to-end (signup → auth → core action → logout)
- Error workflows (invalid input → proper error response → no side effects)
- Cross-service workflows if applicable (API → queue → worker → result)
- Data integrity flows (create → read → update → verify → delete → verify gone)

### 3. CLI tests

Does every CLI command have tests?
- Test both human-friendly output and `--json` output
- Test error cases (missing required flags, invalid input)
- Test that `--json` output is valid JSON and contains expected fields
- CLI is a first-class interface — untested CLI commands are as bad as
  untested API endpoints

### 4. Edge cases

Are boundary conditions tested?
- Empty inputs, null values, max-length strings
- Concurrent access, race conditions
- Permission boundaries (can user A access user B's data?)
- Pagination boundaries (0 items, 1 item, exactly N items, N+1 items)

### 5. Existing test quality audit

Are existing tests actually testing anything?

**Mechanical scan (run first):**
```bash
# Skipped/disabled tests — coverage holes disguised as coverage
grep -rn -e '@pytest\.mark\.skip' -e '@pytest\.mark\.xfail' -e 'pytest\.skip(' \
  -e '\.skip(' -e '\.todo(' -e 'xit(' -e 'xtest(' -e 'xdescribe(' \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
  <scope>

# Trivially passing assertions — fake tests
grep -rn -e 'assert True' -e 'assert 1 ==' -e 'expect(true)\.toBe(true)' \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
  <scope>

# Weak assertions — pass with any stub return
grep -rn -e 'assert .* is not None$' -e 'expect(.*).toBeDefined()$' \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
  <scope>

# Commented-out assertions — someone hid a failure
grep -rn -e '# assert' -e '// expect' -e '// assert' \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
  <scope>

# Excessive mocking — testing mocks, not code
grep -rn -e 'mock\.patch' -e 'Mock(' -e 'MagicMock(' -e 'jest\.mock(' \
  -e 'vi\.mock(' -e 'jest\.spyOn(' \
  --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
  <scope>
```

Then manually verify: are assertions checking specific values or just existence?
A test that asserts `result is not None` passes with any stub return.

## Process

1. Find all source files and their corresponding test files
2. Run the mechanical scans above
3. Quantify the gap: "N modules have no tests, M have partial coverage,
   K existing tests are fake/weak"
4. **Output the Issues Table** (see Output Protocol below)
5. **Walk through issues one-at-a-time** with the user
6. Create beads for each gap using `bd create` with:
   - **Epic** per coverage category (unit tests, e2e, CLI tests, quality fixes)
   - **Tasks** per module or workflow that needs tests
   - **Subtasks** for individual test scenarios within each task
   - What to test and why it's risky without tests
   - Specific test scenarios and edge cases to cover
   - Which test framework and patterns to use (match existing conventions)
   - Dependencies on implementation beads if applicable
   - **Detailed comments** on each bead explaining the testing approach,
     what assertions to make, and what NOT to mock
7. Wire dependencies: test beads block their impl beads (`bd dep add <impl-id> <test-id>`)
8. **Final status table** after all gaps are addressed

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing your analysis, present ONLY a compact table:

```
| #  | Handle              | Description                                        | Crit | Status |
|----|---------------------|----------------------------------------------------|------|--------|
| 1  | no-auth-e2e         | Auth flow has no end-to-end integration test        | High | ✗      |
| 2  | mocked-db-layer     | User repo tests mock the DB — hiding real SQL bugs  | High | ✗      |
| 3  | weak-api-asserts    | API tests only check status 200, not response body  | Med  | ✗      |
```

Column definitions:
- **#**: Sequential number
- **Handle**: 2-5 word slug
- **Description**: 1-2 sentences
- **Crit**: `High` / `Med` / `Low`
- **Status**: `✗` (open) or `✓` (addressed)

Sort by criticality (High first), then by risk (critical paths first).

After presenting the table, say:
> "Ready to walk through each issue. Say **go** to start from #1, or pick a number."

### Phase 2 — One-at-a-Time Walkthrough

For each issue:
1. State the handle and issue number
2. Show the full analysis: what's missing, what's at risk, proposed bead structure
3. Show the `bd create` commands for epics/tasks/subtasks
4. Wait for user approval before creating beads
5. Once resolved or deferred, mark status `✓`
6. Move to the next issue

### Phase 3 — Final Status Table

After all issues walked, present the updated table plus a bead summary:
- Total beads created (epics / tasks / subtasks)
- Dependency structure overview
- Coverage before vs planned coverage after

## Rules

- Every test bead must specify concrete scenarios, not just "add tests for X."
- Subtasks should be granular enough that a single agent can complete one in
  1-2 hours of focused work.
- Match the project's existing test patterns and conventions.
- Prefer real test infrastructure (test DB, HTTP test client, tmp dirs) over mocks.
- Every bead gets detailed comments explaining the testing approach.
- Use extended thinking for coverage analysis.
