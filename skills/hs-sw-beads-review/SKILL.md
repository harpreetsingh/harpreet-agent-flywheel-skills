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
5. **Cold-agent self-sufficiency check** (CRITICAL — run for every bead):

   The test: *Can an agent implement this bead with ONLY the bead description,
   acceptance criteria, and the codebase? No reading other beads, no PLAN.md,
   no prior session context.* If the answer is no, the bead fails.

   For each bead (`bd show <id>`), verify:
   - [ ] **No cross-references** — grep the description for "See PLAN.md",
     "See bead", "as described in", "refer to", "per the plan". These are
     **critical failures**. The referenced content must be inlined verbatim.
     A worker agent cannot read PLAN.md — it only sees the bead.
   - [ ] **Context** — does the description explain WHY this exists, not just
     WHAT to do? A bead that says "Add role field to invite" without explaining
     the feature it serves will produce code that technically works but doesn't
     integrate.
   - [ ] **Type/model definitions inlined** — if the bead mentions TypeScript
     types, Pydantic models, DB schemas, or API contracts, are the actual field
     definitions in the bead? "Create the InviteRequest model" fails.
     "Create InviteRequest with fields: email (str), role (MemberRole enum:
     admin|member|viewer), workspace_id (UUID)" passes.
   - [ ] **Endpoint URLs and methods** — if the bead involves API work, are
     exact routes (`POST /api/v1/invites`), request shapes, and response shapes
     specified? Missing URLs mean the agent invents them.
   - [ ] **Return types, request shapes, event names** — for hooks, SSE, or
     async work: are the actual shapes, event names, and import paths specified?
     "Add real-time updates" fails. "Listen to SSE event `bead:status_changed`
     with payload `{bead_id: string, status: string}` from `/api/v1/events`" passes.
   - [ ] **File pointers** — are relevant files, endpoints, or components named
     with actual paths? "Update the invite dialog" fails. "Update
     `src/components/members/InviteDialog.tsx`" passes.
   - [ ] **Existing code to reuse** — has the bead identified existing components,
     utilities, or patterns the worker should extend rather than reinvent? If not,
     grep/glob for relevant concepts and add pointers: "Use existing InviteDialog
     in `src/components/members/InviteDialog.tsx` as the base — extend with role
     prop." This is CRITICAL for brownfield codebases.
   - [ ] **UX baseline** (frontend beads) — does the bead specify which existing
     UX patterns to follow? ("Match the member list pattern in MemberTable.tsx —
     same row height, same action menu position, same empty state.") Without this,
     workers invent their own patterns and create visual inconsistency.
   - [ ] **Mock/test data shapes** — if the bead or its test-pair needs fixtures,
     are the shapes specified? "Write tests with mock data" fails. "Mock
     `listInvites` returning `[{id: uuid, email: string, role: MemberRole,
     status: 'pending'|'accepted'}]`" passes.
   - [ ] **Scope boundary** — does the bead explicitly state what's OUT of scope?
     For frontend beads: "Do NOT modify layout/spacing/styling of existing
     components."
   - [ ] **Execution steps present** — does the bead description include a
     `## Steps` section with the 4 standard steps (Search, Read, Implement,
     Verify)? The Search step must include all three layers:
     1. `cm context` — prior rules and anti-patterns from CM playbook
     2. `cass search` — past sessions that solved similar problems
     3. Serena + grep/glob — existing code in the current codebase
     If missing or incomplete: add the standard steps template. Required
     for any bead that will be worked in an upcoming wave.
   - [ ] **Mechanically verifiable AC** — every acceptance criterion must be
     checkable by grep, curl, test output, or file read. If a criterion says
     "it works" or "handles errors" — rewrite it with the specific observable
     (endpoint returns 200 with `{id, status}`, component renders role dropdown
     with 3 options, `pytest tests/test_invite.py` passes).
   - [ ] **Negative-path ACs (always-on components)** — if the bead delivers a
     loop, worker, lifespan task, or other always-on/runtime component: it must
     carry ACs (each testable) for (a) required config absent → degrade gracefully,
     log once, no per-tick spam; (b) failure before task/lease creation → no leaked
     permits, claimed item reaches a terminal state; (c) connection drop → reconnect.
     Happy-path-only hardening beads are a known escape source — flag High.
   - [ ] **Real-callee signature (interface-wiring beads)** — if the bead wires a
     caller to an EXISTING callee (service/resolver/client/cursor): the bead must
     inline the callee's ACTUAL signature (verified against source, not assumed)
     and require a contract test importing the real symbol. An assumed interface
     (async-vs-sync, dict-vs-tuple cursor) is the top contract-drift escape — flag High.

   **If a bead fails any check:** flag it as an issue. Propose the enriched
   description with the missing content inlined.

6. **Right-sizing check** (CRITICAL — mechanical, run for every bead):

   A bead can pass every self-sufficiency check and still be the wrong SIZE — and
   oversized beads are the #1 predictor of rework (one agent botches a sprawling
   task). Self-sufficiency asks "has enough context"; this asks "is it the right
   size to finish in one shot without rework." Use mechanical thresholds, not gut feel:

   **Flag OVERSIZED → SPLIT if any of:**
   - [ ] **Spans >1 architectural layer** — touches DB/schema AND API AND/OR UI in
     one bead. Split into one bead per layer, with an interface contract between
     them (the producer bead defines the signature/schema, the consumer quotes it).
     Cross-layer beads are where parallel agents drift most.
   - [ ] **>5 acceptance criteria** — it's doing too many things. Split by deliverable.
   - [ ] **>~5 files in its file pointers** (or no file pointers but obviously broad
     scope) — too large to hold in one agent's head. Split.
   - [ ] **More than one distinct deliverable** in the title ("X and Y") — split on the "and".

   **Flag TOO GRANULAR → MERGE if:**
   - [ ] Single trivial AC, <30 min of work, no test pairing warranted — merge into
     its natural sibling bead to avoid orchestration overhead.

   For each flagged bead, propose the concrete split/merge (new titles + the
   dependency/contract edges) in the Issues Table.

7. **Contract consistency check** (cross-bead — the deepest anti-drift check):

   Self-sufficiency (step 5) verifies each bead inlines its shapes; this verifies
   that producer and consumer inlined the *same* shape. Divergent contracts across
   parallel beads are Yegge's "specs drift silently" — the integration break that
   no single-bead check can catch.

   For each producer→consumer dependency edge that crosses an interface (API route,
   type/model, event, DB schema):
   - [ ] **Both beads carry a `## Contract` block** for the shared interface. If
     the producer or a consumer is missing it → flag High, add it.
   - [ ] **The blocks are byte-identical** — grep the literal signature/shape
     string across the pair (`bd show <A>`, `bd show <B>`). Any difference in field
     names, types, route, method, or response shape → flag High. The producer
     (source of truth) wins; update the consumer to match.
   - [ ] **Source of truth is named** — the contract block names the producing
     bead ID, so there's no ambiguity about who owns the shape.
   - [ ] **Both directions for wiring beads** — a contract that pins the response
     shape but assumes the request shape (or vice versa) is HALF a contract. The
     request model's declared fields AND the response serializer's emitted keys
     must both be pinned against the REAL symbols. (GH#342: an empty request
     model silently discarded a field the CLI sent; a serializer omitted the new
     column the CLI read — both passed every single-sided check.)

7b. **Transcript-vs-AC diff** (the e2e/persona bead vs the implementing beads —
   GH#342 retro's top correction):

   If the set contains an e2e/live-fire/persona bead with a command transcript or
   journey: walk EVERY step of it against the implementing beads' contracts.
   - [ ] **Every transcript step names its implementing bead.** A step no bead
     implements is a planning gap that will surface as a live-fire escape
     (GH#342: `plan --new` was transcript prose with no implementing AC; the
     no-`--mode` stage-default step had no bead teaching the CLI to omit the key).
   - [ ] **Every flag, argument form, and expected output line in the transcript
     exists in some implementing bead's AC or contract** — byte-compare the
     command forms, not the vibes.
   - [ ] **Seam steps get seam coverage**: where step N's state feeds step N+1's
     default behavior (e.g. a transition then an omitted-flag run), some bead's
     AC must test that interaction, not just each side.
   Divergences are High — resolve before W1 (fix the transcript or add/amend the
   implementing bead).

8. **File-overlap graph** (collision-free scheduling input):

   Two beads that touch the same file are *coupled* even when the dependency graph
   says they're independent — that hidden coupling is where parallel agents collide.
   Surface it now so exec-plan + the Director can schedule collision-free.

   Read every bead's `## Files` section and build the overlap graph (edge = shared
   path; be glob-aware — `src/auth/**` overlaps `src/auth/api.py`):
   - [ ] **Every bead HAS a `## Files` section** — missing = flag, the bead is
     unschedulable for collision detection. Add it.
   - [ ] **Two beads both `create` the same path** → flag **High**: almost always a
     decomposition error (two beads can't both own a new file). Merge or re-split.
   - [ ] **Overlap with NO logical dependency edge** → flag: these can't safely run
     in parallel. Propose one of: (a) **co-assign** — group them onto one worker
     lane (the worker holds the shared file start-to-finish, zero collision); or
     (b) **serialize** — add a dependency edge so one finishes before the other.
   - [ ] **Emit the overlap graph** in the Issues Table / final output so
     `/hs-sw-sprint-exec-plan` can consume it for parallel-width + lane assignment.

9. **Deep review** (for each bead, `bd show <id>`):
   - Does this bead make sense? Is it necessary?
   - Are dependencies correct and complete?
   - Could beads be reordered (wrong priority) or removed (unnecessary)?
10. **Output the Issues Table** (see Output Protocol below) — do NOT elaborate yet
11. **Walk through issues one-at-a-time** with the user (see Output Protocol)
12. **Apply revisions** during walkthrough using `bd update` and `bd dep add`
   after user approves each change
13. **Final status table** after all issues are walked

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing all checks (steps 1-6), present ONLY a compact table. No elaboration,
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
- Self-sufficiency failures found and fixed (cross-references, missing types, missing URLs)
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
