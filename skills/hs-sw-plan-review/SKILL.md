---
name: hs-sw-plan-review
description: Deeply review a markdown plan and propose concrete improvements with rationale and diffs
argument-hint: [plan-file-path]
---

# /plan-review — Iterative Plan Improvement

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → ★REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Iterative refinement. 4-5 rounds until convergence.     │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Read the plan file at `$ARGUMENTS` (default: `PLAN.md`) and perform one round of
deep review.

## What to do

1. Read the entire plan file carefully. Also check for a `planning-context/`
   directory alongside the plan — if it exists, read the evidence (research,
   competitor analysis, PRDs, reference material). This grounds your review
   in the same inputs that informed the plan.
2. Evaluate it holistically for:
   - Architecture quality, robustness, and simplicity
   - Missing, weak, or unnecessary features
   - Performance, reliability, and security gaps
   - How compelling and useful it is for end users
   - Feasibility and implementation cost of each section
   - **Prior art and brownfield awareness** — does the plan show evidence of
     checking CM, CASS, and the codebase? Verify:
     1. Did the planner run `cm context` and incorporate relevant rules/anti-patterns?
     2. Did the planner run `cass search` and reference prior sessions that touched
        this area? Past debugging sessions and design decisions are gold.
     3. Does the "What Exists Already" section have file paths verified via
        Serena + grep — not just assertions?
     Flag any section that proposes building something new without checking all
     three layers. For maturing codebases, the plan MUST include a "What Exists
     Already" section mapping prior sessions AND existing code to planned features.
     Missing this causes agents to reinvent during sprints.
   - **CLI completeness** — does every feature with an API or UI also specify
     CLI commands? Are `--json` flags included? Is the CLI a first-class
     interface or an afterthought? Flag any feature missing CLI commands as
     a High-severity gap.
   - **Testing strategy** — does the plan have a "Testing Strategy" section?
     Flag a missing testing strategy as High-severity — it's the root cause
     of untested code and stubs surviving sprints. When present, verify it covers:
     - **Unit tests without mocks** — which modules need unit tests? Plan must
       state that mocks are only acceptable at system boundaries (HTTP clients,
       database drivers, external APIs) — never for internal functions.
     - **E2e integration tests** — which user workflows get end-to-end tests?
       Plan must specify detailed logging requirements (every step logs what
       it does, every assertion logs expected vs actual, failures include full
       context).
     - **CLI tests** — every CLI command tested for human + `--json` output?
     - **What's real vs mocked** — explicit list of what uses real test
       infrastructure (test DB, HTTP test client) vs what gets mocked and why.
     - A vague "we'll add tests" section is as bad as no section. The plan
       must name specific modules, workflows, and test approaches.
3. **Output the Issues Table** (see Output Protocol below) — do NOT elaborate yet
4. **Walk through issues one-at-a-time** with the user (see Output Protocol)
5. For each issue during walkthrough, provide:
   - **Analysis**: What's wrong or suboptimal
   - **Rationale**: Why the change makes it better — with specifics, not hand-waving
   - **Diff**: git-diff style change relative to the current plan
6. Do NOT apply changes. Wait for user approval on each issue.
7. **Final status table** after all issues are walked

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing your evaluation, present ONLY a compact table. No analysis, no diffs,
no paragraphs of rationale. Just the table:

```
| #  | Handle           | Description                                      | Crit | Status |
|----|------------------|--------------------------------------------------|------|--------|
| 1  | missing-test-strat | No testing strategy section — root cause of stubs | High | ✗      |
| 2  | vague-ac         | Acceptance criteria in D3 are not verifiable       | Med  | ✗      |
```

Column definitions:
- **#**: Sequential number
- **Handle**: 2-5 word slug that makes the issue easy to reference in conversation
  (e.g., `missing-test-strat`, `scope-creep-d4`, `no-cli-surface`)
- **Description**: 1-2 sentences, no more
- **Crit**: `High` / `Med` / `Low`
- **Status**: `✗` (open) or `✓` (addressed)

Sort by criticality (High first), then by plan section order.

After presenting the table, say:
> "Ready to walk through each issue. Say **go** to start from #1, or pick a number."

### Phase 2 — One-at-a-Time Walkthrough

For each issue (in order, or as the user picks):
1. State the handle and issue number
2. Show the full analysis, rationale, and proposed diff
3. Wait for the user to approve, reject, or modify before moving on
4. Once resolved or deferred, mark status `✓` or note deferral
5. Move to the next issue

Do NOT present multiple issues at once. One issue per response.

### Phase 3 — Final Status Table

After all issues have been walked, present the updated table with final statuses.
If any remain `✗`, call them out explicitly and ask if the user wants another pass.
When suggestions become very incremental across rounds, tell the user:
"This plan has likely converged — ready for /beads-create."

## Convergence

Each invocation is one round. After 4-5 rounds the suggestions become very
incremental. Convergence is signaled in the Phase 3 final status table.

## Rules

- Be bold. Propose real architectural changes, not cosmetic tweaks.
- Every suggestion must have a concrete rationale — no "consider doing X" without why.
- Preserve the plan's voice and structure while improving substance.
- Do NOT apply changes automatically. Present them for approval.
- Use extended thinking for deep analysis.
