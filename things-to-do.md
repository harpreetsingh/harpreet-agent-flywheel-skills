# Things To Do — Agent Flywheel Improvements

Current rating: **8 / 10** for producing high-quality code hands-free.

## What keeps it from 9-10

### 1. No visual verification for frontend
Agents can't see what they build. UX review is code-level — grep for patterns,
check component states. A stub that renders a blank `<div>` with the right class
names passes every mechanical check.

**Fix:** Screenshot/visual diffing via headless browser snapshots. CM, CAAS
starts addressing this — agents that can see and interact with the running UI.

### 2. No automated runtime testing during sprint
QA smoke tests check endpoints *if servers happen to be running* but nothing
starts them. For hands-free, we need: start servers → hit endpoints → verify
responses → shut down.

**Fix:** Automated server lifecycle in smoke tests. CM, CAAS addresses this —
agents can start processes, wait for readiness, and interact with running services.

### 3. Context window is the real ceiling
A Director managing a 3-wave sprint with 5 workers, QA, wave gates, review
flywheels, mid-wave deep review will compact. Recovery exists but is inherently
lossy. System quality degrades as sprint complexity grows.

**Fix:** Tighter checkpoint protocol, smaller wave sizes, or Director delegation
(sub-directors per wave). Long-term: unlimited reliable context.

### 4. No cross-sprint learning
`lessons.md` gets generated but never fed back. Sprint N+1 doesn't benefit from
sprint N's failures.

**Fix:** Feedback loop from QA failure patterns → Phase 0 enrichment. Auto-memory
of common failure modes per project. Sprint retrospective that updates skill
parameters (e.g., "this codebase has a pattern of missing error handling in API
routes — add to Phase 0 checklist").

### 5. Grep-based detection has a ceiling
A function that calls the real API but ignores the response and returns a
hardcoded value passes every mechanical scan. The reasoning-level reviews (bug
hunter, peer reviewer) are the backup, but they're probabilistic, not guaranteed.

**Fix:** Runtime assertion verification — actually call the functions and verify
return values match expected behavior, not just pattern-match the source. CM, CAAS
helps here too — agents can run code and observe results.

## Maturity tiers

### To reach 9/10
- [ ] Automated server lifecycle in smoke tests (start → test → stop)
- [ ] Screenshot/visual diffing for frontend (headless browser snapshots)
- [ ] Sprint retrospective memory that feeds into future Phase 0 enrichment
- [ ] CM, CAAS integration for visual verification and runtime interaction

### To reach 10/10
- [ ] Agents that can see and interact with the running UI (CM, CAAS)
- [ ] Unlimited reliable context (no compaction loss)
- [ ] Self-improving: system detects its own failure patterns and patches its own skills
- [ ] Cross-sprint learning loop (failure patterns → skill updates → prevention)

## What's already exceptional (the 8)

For reference — what's working well and should be preserved:

- **Anti-stub/anti-fake defenses** — 4+ layers catching hollow code (mandate →
  self-review → mechanical grep → QA reality checks → wave gate scans)
- **Structurally enforced TDD** — different workers write tests vs implementation,
  dependency-blocked, QA-verified, mechanically scanned for fraud
- **Brownfield discipline** — 5-point enforcement chain preventing code
  reinvention and silent UX drift
- **Review convergence** — calibrated at multiple levels (plan 4-5 rounds, deep
  review 2 clean, wave gate capped at 3)
- **Recovery from compaction** — checkpoints, lifecycle beads, recovery protocol
- **Separation of concerns** — Director/QA/Bug Hunter/Peer Reviewer/Workers each
  have clear, non-overlapping responsibilities
- **Security baked in** — 4-layer defense (worker, wave gate UBS, bug hunter
  reasoning, sprint close full scan)
