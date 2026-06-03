---
name: hs-sw-fresh-eyes
description: Cold-read audit of any artifact — code, plans, docs, research, or beads — for bugs, gaps, and problems
argument-hint: [file-or-directory-or-bead-id]
---

# /fresh-eyes — Fresh Eyes Review

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE   │
│ ★ YOU ARE HERE: Sprint close + anytime. Cold-read audit of anything.    │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Cold-read any artifact as if you've never seen it. Find bugs, gaps, contradictions,
and problems. Fix what you can, report what you can't.

Works on: **code, PLANs, research/planning-context, docs, beads.**

## Scope

- If `$ARGUMENTS` is a file or directory → review those artifacts
- If `$ARGUMENTS` is a bead ID (e.g., `beads-xxx`) → review that bead and its dependents
- If `$ARGUMENTS` is `--beads` → review all open beads (`bd list --status=open`)
- If `$ARGUMENTS` is `--self-review` → **post-implementation self-review** (see below)
- If no arguments → detect what changed in this work session:
  1. Try `git diff --name-only $(git merge-base HEAD main)..HEAD`
  2. Fallback to `git diff --name-only HEAD~5` if on main

## Self-Review Mode (`--self-review`)

Lightweight mode for after you've just finished implementing something. Faster
and more targeted than a full cold-read audit. Use this before reporting a
ticket as done, or anytime you've just written code.

**Scope:** only files you created or modified (from `git diff --name-only` of
uncommitted + staged changes).

**Process — run in rounds until clean:**

**Each round:**
1. Get the list of changed files: `git diff --name-only HEAD` + `git diff --name-only --cached`
2. Read every changed file completely — no skimming
3. For each file, look specifically for:
   - **Bugs:** logic errors, off-by-one, wrong operators, null handling, missing returns
   - **Stubs:** placeholder code you meant to come back to but didn't
   - **Half-implementations:** happy path only, missing error cases, switch missing branches
   - **Copy-paste artifacts:** wrong variable names, stale comments from copied code
   - **Contract violations:** does the code match what callers/tests expect?
   - **Dead code:** unused imports, assigned-but-never-read variables
4. Run the mechanical stub scan (same grep from the Code checklist)
5. Run the mechanical test integrity scan if any test files were changed
6. Fix everything found
7. If fixes were made → run another round (bugs cluster — fixing one often reveals another)
8. If no fixes needed → clean. Stop.

**Convergence:**
- Simple code: typically 1-2 rounds
- Complex code: typically 2-3 rounds
- If still finding bugs after 3 rounds: stop and escalate. The implementation
  approach may be fundamentally off — a full `/fresh-eyes` review or a different
  implementer may be needed.

**Report:** `Self-review: DONE — N rounds, M total issues fixed`
(e.g., `Self-review: DONE — 2 rounds, 4 issues fixed` or `Self-review: DONE — 1 round, clean`)

**This mode skips** the full Output Protocol (no issues table, no walkthrough).
It's fast iterative passes: scan, fix, repeat until clean. If you find complex
architectural issues, escalate to a full `/fresh-eyes` review.

## Auto-Detect Artifact Type

Determine what you're reviewing and apply the matching lens:

| Input | Detected type |
|-------|--------------|
| `*.py`, `*.ts`, `*.tsx`, `*.js`, source dirs | **Code** |
| `PLAN.md`, `**/PLAN*.md` | **Plan** |
| `pitch.md` | **Pitch** |
| `planning-context/`, research docs | **Research** |
| `docs/projects/features/**/architecture.md`, `api.md`, `cli.md`, etc. | **Internal Docs** |
| `docs/areas/site/**` | **External Docs** |
| `beads-xxx`, `--beads` | **Beads** |
| Mixed directory (e.g., `docs/projects/features/org-management/`) | **All types found** — run each lens |

## Process

1. Identify artifacts and their types
2. Read each artifact completely — no skimming
3. Apply the matching checklist (below)
4. Classify each finding: **Bug** / **Defect** / **Gap** / **Nit**
5. **Output the Issues Table** (see Output Protocol below) — do NOT elaborate yet
6. **Walk through issues one-at-a-time** with the user (see Output Protocol)
7. For code and docs: fix Bugs and Defects directly during walkthrough.
   For Plans, Pitches, and Beads: report findings — do NOT edit without approval.
8. **Final status table** after all issues are walked

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing your scan, present ONLY a compact table. No elaboration, no diffs,
no paragraphs of analysis. Just the table:

```
| #  | Handle           | Description                                      | Crit | Status |
|----|------------------|--------------------------------------------------|------|--------|
| 1  | creation-rail    | The rail on UX requires better definition        | High | ✗      |
| 2  | icon-broken      | The icons on the UX needs revisiting             | Low  | ✗      |
```

Column definitions:
- **#**: Sequential number
- **Handle**: 2-5 word slug that makes the issue easy to reference in conversation
  (e.g., `missing-error-boundary`, `stale-api-ref`, `vague-ac`)
- **Description**: 1-2 sentences, no more
- **Crit**: `High` / `Med` / `Low`
- **Status**: `✗` (open) or `✓` (addressed)

Sort by criticality (High first), then by artifact order.

After presenting the table, say:
> "Ready to walk through each issue. Say **go** to start from #1, or pick a number."

### Phase 2 — One-at-a-Time Walkthrough

For each issue (in order, or as the user picks):
1. State the handle and issue number
2. Show the full analysis: what's wrong, where, why it matters
3. For code/docs: propose or apply the fix. For plans/beads: propose the fix and wait.
4. Once resolved or deferred, mark status `✓` or note it was deferred
5. Move to the next issue

Do NOT present multiple issues at once. One issue per response.

### Phase 3 — Final Status Table

After all issues have been walked, present the updated table with final statuses.
If any remain `✗`, call them out explicitly and ask if the user wants another pass.

---

## Checklists by Artifact Type

### Code

- **Stub/fake scan (run first, mechanical):**
  ```bash
  grep -rn -e 'TODO' -e 'FIXME' -e 'HACK' -e 'XXX' -e 'placeholder' \
    -e 'NotImplementedError' -e '^\s*pass$' -e 'stub' -e 'dummy' \
    -e 'mock.*implementation' -e 'fake.*response' -e 'hardcoded.*return' \
    --include="*.py" --include="*.ts" --include="*.tsx" \
    <scope> | grep -v test | grep -v __pycache__
  ```
  Then manually verify: functions ≤3 lines with complex responsibilities,
  imports present but never called, arguments received but ignored, empty
  catch blocks silently swallowing errors, hardcoded return values
- **Test integrity scan (run second, mechanical):**
  ```bash
  grep -rn -e '@pytest\.mark\.skip' -e '@pytest\.mark\.xfail' -e 'pytest\.skip(' \
    -e '\.skip(' -e '\.todo(' -e 'xit(' -e 'xtest(' \
    -e 'assert True' -e 'assert 1' -e '# assert' -e '// expect' \
    --include="*.py" --include="*.ts" --include="*.tsx" --include="*.test.*" \
    <scope>
  ```
  Also check: are assertions testing specific values or just existence/truthiness?
  Tests that only assert `is not None` pass with any stub return.
- Logic errors, off-by-one, wrong comparison operators
- Null/undefined handling, missing return statements
- Type mismatches, incorrect async/await, unhandled promise rejections
- Race conditions, missing error handling at system boundaries
- Typos in strings/keys/URLs, wrong variable names
- Copy-paste artifacts, stale references after a refactor
- API contract violations (caller doesn't match callee signature)
- Security: injection, auth bypass, data exposure (OWASP top 10)
- Dead code, unused imports, unnecessary abstractions

### Plan (PLAN.md)

- Does the problem statement match what the solution actually solves?
- Are there deliverables described in prose that should be a diagram? (paragraphs describing flows = red flag)
- Are there diagrams? Every non-trivial plan needs at least one ASCII diagram.
- CLI commands specified for every API/UI feature? `--json` included?
- Testing strategy section present? Identifies which deliverables need tests,
  what kind, key scenarios, and mock boundaries?
- Acceptance criteria: are they concrete and verifiable, or vague?
- Scope: anything that looks in-scope but isn't explicitly listed?
- Dependencies between deliverables: are they stated? Any missing?
- Feasibility vs appetite: does the work actually fit the stated time budget?
- Contradictions between sections (e.g., solution says X, deliverables say Y)
- Stale references to things that were renamed, removed, or reorganized

### Pitch (pitch.md)

- Is the problem a concrete user situation, not an abstraction?
- Does the appetite feel right for the described scope?
- Is the solution at fat-marker level, or over/under-specified?
- Do rabbit holes have explicit decisions (in/out/defer)?
- Are no-gos explicit, not implied?
- Could a senior engineer start building without asking clarifying questions?
- CLI surface specified?

### Research / Planning Context

- Are sources cited or is it unsupported assertion?
- Contradictions between documents (one doc says X, another says Y)
- Stale information (dates, version numbers, API references that may have changed)
- Missing perspectives: does the research only look at one competitor/approach?
- Conclusions that don't follow from the evidence presented
- Key questions the research raises but doesn't answer
- Relevance: is everything here actually informing the plan, or is some of it noise?

### Internal Docs (architecture.md, api.md, cli.md, data-model.md, etc.)

- Does the architecture doc match what the code actually does? (read the code to verify)
- API docs: do endpoints, request/response schemas match the actual implementation?
- CLI docs: do commands, flags, `--json` schemas match what's implemented?
- Data model: do tables, relationships, constraints match the actual schema?
- Are diagrams present and accurate?
- Cross-references: do links to other docs resolve?
- Stale content: anything describing behavior that was changed or removed?
- Gaps: is there implemented functionality with no documentation?

### External Docs (docs/areas/site/)

- Can an external reader understand this without internal context?
- Undefined jargon or internal terminology?
- Are examples complete and runnable (not pseudocode)?
- Do CLI examples include both human-friendly and `--json` output?
- Do API examples include request AND response?
- Are error scenarios documented?
- Frontmatter present and correct (title, description, category, feature)?
- Gaps: features that exist but have no external documentation?

### Beads

For each bead (`bd show <id>`):
- **Title:** descriptive enough to understand without reading description?
- **Description:** explains WHY, not just WHAT?
- **Acceptance criteria:** concrete and verifiable? (not "it works", "it's good")
- **Dependencies:** correct? Missing any? Circular?
- **Scope:** clear what's in and out?
- **File pointers:** are relevant files, endpoints, components named?
- **TDD pairing:** does this impl bead have a companion test bead?
- **Stale:** does this bead reference things that have been renamed/removed?
- **Duplicate:** is this bead's scope overlapping with another bead?
- **Size (mechanical):** flag OVERSIZED if it spans >1 architectural layer
  (DB+API+UI), has >5 acceptance criteria, names >~5 files, or bundles >1
  deliverable ("X and Y") — propose a split. Flag TOO GRANULAR if it's a <30-min
  trivial change with no test pairing — propose a merge.

---

## Rules

- Approach every artifact as if you've never seen it before — that's the whole point.
- Don't rationalize away suspicious content. If it looks wrong, investigate.
- For code and docs: fix Bugs and Defects directly. Report Nits.
- For plans, pitches, and beads: report ALL findings. Do NOT edit without approval.
- When reviewing a feature directory, check ALL artifact types present — don't stop at code.
- Comply with all rules in CLAUDE.md and AGENTS.md.
- Use extended thinking for complex analysis.
