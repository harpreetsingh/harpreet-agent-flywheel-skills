# Things To Do — Agent Flywheel Improvements

Current rating: **8 / 10** for producing high-quality code hands-free.

## What keeps it from 9-10

### 1. No visual verification for frontend
Agents can't see what they build. UX review is code-level — grep for patterns,
check component states. A stub that renders a blank `<div>` with the right class
names passes every mechanical check.

**Fix:** Screenshot/visual diffing via headless browser snapshots. CM, CAAS
starts addressing this — agents that can see and interact with the running UI.

#### Research: How to Actually Fix This (2025-05-18)

Two distinct failure modes observed in practice:

1. **Agents don't know where UX changes land** — no component tree map at task
   time, so new components get placed arbitrarily.
2. **Destructive rewrites** — agents rewrite entire component files instead of
   making surgical edits. This is a documented 2025 model regression (~2x more
   full-file rewrites after model updates). Correct result via scorched-earth
   rewrite is still a failure.

**Tooling that exists today:**

- **Playwright MCP** — the emerging standard. Gives Claude 25+ structured tools
  (navigate, click, type, screenshot, DOM query) via the accessibility tree.
  10-100x faster/cheaper than vision-based screenshot guessing. Cross-browser.
  Setup: ~10 min. Install `@playwright/mcp`, add to MCP config.
- **`claude --chrome`** (Chrome extension + Claude Code) — inherits your existing
  browser login session; no auth harness needed. High context usage, so use only
  for verification passes, not during implementation.
- **Computer use** — sees and controls the full desktop (macOS/Linux). Slower
  than Playwright MCP; best for native apps or things outside the browser.
- **Visual regression baselines** — Percy, Chromatic, or committed PNGs.
  Playwright MCP can verify the happy path; baselines catch regressions.

**Credential/auth pattern:**
Specify the test account identity in AGENTS.md (username + role). Inject the
actual credential via env var or JIT token at session start — never hardcode
plaintext credentials in bead text (prompt injection risk via on-screen content).

**What needs to change in this system:**

1. **Component map** — Add `component-map.md` to each project's AGENTS.md:
   which component owns which UI region, which files are authoritative per
   section. Without this agents guess placement.

2. **Surgical edit mandate in AGENTS.md** — Explicit rule: never rewrite a
   component file wholesale. Read the file, identify the single function/section
   to change, change only that. This must be at the instruction layer, not left
   to bead-level hope.

3. **Frontend bead template additions** — Every frontend bead must include:
   exact component file path, target section/function, which existing component
   states must be preserved (loading/empty/error/data/partial — all 5), and an
   explicit do-not-touch list for adjacent components.

4. **QA diff-scope check** — QA agent should verify the diff touches *only* the
   files/sections specified in the bead, not just that the result is correct.

5. **Playwright MCP in QA smoke tests** — Replace or augment curl-based smoke
   tests with Playwright MCP for frontend tickets. QA agent gets real DOM
   verification, not just HTTP response codes.

**When to use which tool:**
- Playwright MCP → web app UI verification (structured, fast, default choice)
- `claude --chrome` → authenticated workflows, live debugging during development
- Computer use → native apps, simulators, anything outside the browser
- Visual baselines → regression detection between sprints

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

### 6. ~~No cross-session memory~~ — covered by CM

**Resolved.** The memory stack is complete:

| Layer | Tool/Skill | Scope |
|-------|-----------|-------|
| 1 | `/stash` + `/hydrate` | Within-session — survives compaction |
| 2 | CM (`cm context`, `cm reflect`, `cm playbook`) | Across-session — curated rules + history from past sessions |
| 3 | auto memory (`memory/` dir) | Across-session — preferences and project context |

`cm context "<task>" --json` already pulls cross-session rules and history before
any task. `cm reflect` extracts patterns from past sessions into the playbook.
A separate `/recall` skill would just wrap what agents should already be doing
via CM directly.

---

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
