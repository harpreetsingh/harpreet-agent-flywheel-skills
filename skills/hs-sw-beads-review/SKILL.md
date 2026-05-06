---
name: hs-sw-beads-review
description: Review and optimize existing beads for correctness, completeness, and structure
---

# /beads-review — Beads Optimization

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → ★DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: QA the beads before sprint. Fix structure, not code.    │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Check over each bead super carefully — are they correct? Optimal? Could anything
be changed to make the system work better for users?

## Process

1. **Structural review first** (cheap, catches big issues):
   - `bd list --status=open` to see all open beads
   - `bd graph --all` to see dependency structure
   - Look for: orphaned beads (no deps, not a root epic), circular dependencies,
     overly long dependency chains, beads with no dependents that aren't leaf tasks
2. **TDD compliance check**:
   - For each impl bead with testable acceptance criteria: does a companion test
     bead exist?
   - Does the test bead BLOCK the impl bead? (`bd show <id>` — check dependencies)
   - Does the test bead's acceptance criteria require tests to FAIL (red phase)?
   - If missing: create the test bead and wire the dependency
   - Flag any impl bead that has no test coverage rationale
3. **Domain balance check**:
   - Count beads by domain (backend, frontend, infra, tests)
   - Flag if any domain has impl beads but zero test beads
   - Flag if frontend and backend both exist but only one has test coverage
   - Flag if >30% of beads are in one domain with no dedicated worker planned
4. **CLI coverage check**:
   - For each impl bead that delivers an API endpoint or UI feature: does it
     also specify CLI commands?
   - Do the CLI commands include `--json` flag for agent/machine consumption?
   - If missing: add CLI requirements to the bead's acceptance criteria
   - Flag any bead that delivers an API/UI without CLI as incomplete
5. **Deep review** (for each bead, `bd show <id>`):
   - Does this bead make sense? Is it necessary?
   - Is the scope right? Too big = hard to execute. Too granular = overhead.
     A good task is 1-4 hours of focused agent work.
   - Does it have **mechanically verifiable** acceptance criteria? Each
     criterion must be checkable by grep, curl, test output, or file read.
     If a criterion says "it works" or "handles errors" — rewrite it with
     the specific observable (endpoint returns X, component renders Y,
     command outputs Z). Vague criteria are the root cause of stubs passing QA.
   - Are dependencies correct and complete?
   - Is the description self-contained enough for an agent with no prior context?
   - Could beads be merged (overlapping scope), split (too large), reordered
     (wrong priority), or removed (unnecessary)?
6. **Output the Issues Table** (see Output Protocol below) — do NOT elaborate yet
7. **Walk through issues one-at-a-time** with the user (see Output Protocol)
8. **Apply revisions** during walkthrough using `bd update` and `bd dep add`
   after user approves each change
9. **Final status table** after all issues are walked

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing all checks (steps 1-5), present ONLY a compact table. No elaboration,
no `bd update` commands, no paragraphs of analysis. Just the table:

```
| #  | Handle           | Description                                      | Crit | Status |
|----|------------------|--------------------------------------------------|------|--------|
| 1  | missing-test-bead | Auth impl bead has no companion test bead        | High | ✗      |
| 2  | vague-ac-signup  | Signup bead AC says "it works" — not verifiable   | Med  | ✗      |
```

Column definitions:
- **#**: Sequential number
- **Handle**: 2-5 word slug that makes the issue easy to reference in conversation
  (e.g., `missing-test-bead`, `orphan-infra-bead`, `oversized-auth`)
- **Description**: 1-2 sentences, no more
- **Crit**: `High` / `Med` / `Low`
- **Status**: `✗` (open) or `✓` (addressed)

Sort by criticality (High first), then by dependency order.

After presenting the table, say:
> "Ready to walk through each issue. Say **go** to start from #1, or pick a number."

### Phase 2 — One-at-a-Time Walkthrough

For each issue (in order, or as the user picks):
1. State the handle and issue number
2. Show the full analysis: what's wrong, why it matters, proposed fix
3. Show the exact `bd update` / `bd dep add` commands you'll run
4. Wait for user approval before executing
5. Once resolved or deferred, mark status `✓` or note deferral
6. Move to the next issue

Do NOT present multiple issues at once. One issue per response.

### Phase 3 — Final Status Table

After all issues have been walked, present the updated table with final statuses.
Organize a summary by category:
- TDD gaps found and fixed
- Domain balance issues
- CLI coverage gaps found and fixed
- Structural changes (merges, splits, reorders)

If any remain `✗`, call them out explicitly and ask if the user wants another pass.

## Rules

- It's cheaper to fix things in plan space than during implementation.
- Be opinionated — propose real structural changes, not just wordsmithing.
- Every bead must have acceptance criteria. If one is missing them, add them.
- Every impl bead with testable criteria must have a companion test bead that
  blocks it. No exceptions — if tests can be written, they must be planned.
- Use extended thinking for structural analysis.
