---
name: hs-sw-sprint-go
description: Launch multi-agent sprint from execution plan — spawn director who owns the team
argument-hint: [--dry-run]
---

# /sprint-go — Launch Sprint

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → ★EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Launch. Spawns Director, walk away. --dry-run to preview│
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Reads the persisted sprint plan from `/hs-sw-sprint-exec-plan` and launches
a multi-agent sprint. This is the walk-away moment.

**Key architecture (revised 2026-07-05):** The LAUNCHER spawns the ENTIRE topology —
Director, workers, QA, and standing reviewers — as named background teammates (teams
form implicitly; `TeamCreate` was removed in v2.1.178). The Director coordinates
exclusively via SendMessage; it never spawns.

> **Why:** spawned subagents cannot spawn subagents in this harness (nesting cap), so
> a Director-creates-team design silently degrades to a solo-sequential sprint (observed
> GH#360, 2026-06-09). The historical reason Director-creates-team existed — worker
> questions up-leveling to the human — is now solved differently:
> 1. Every worker/QA/reviewer spawn prompt names the Director as its manager:
>    "Route ALL questions, blockers, and reports to `director` via SendMessage.
>    Never address the human."
> 2. Every spawn sets an explicit permission `mode` (e.g. `acceptEdits` or
>    `bypassPermissions` per the sprint's autonomy mandate) so tool-permission
>    prompts do not bubble to the human session.

## Dry-Run Mode

If `$ARGUMENTS` contains `--dry-run`: simulate the entire sprint without spawning
any agents, creating any teams, or modifying any beads. This is a preview of what
the Director would do.

### Dry-Run Process

1. Read `tmp/sprint-exec-plan.md` (same as live mode)
2. Read all beads referenced in the plan (`bd show <id>` for each)
3. **Simulate the full Director lifecycle** and output a narrative:

```markdown
## Sprint Dry-Run: <feature-name>

### Team
- Director (fable) — coordinator
- Worker-1 (backend): T-01, T-04, T-07
- Worker-2 (frontend): T-02, T-05, T-08
- QA-1: all tickets

### Phase 0 — Ticket Sufficiency
[For each bead: PASS or NEEDS ENRICHMENT with what's missing]

### Wave 1 — Foundation (estimated: X tickets)
1. **Assign:** T-01-test → Worker-1 (red phase), T-02-test → Worker-2 (red phase)
2. **QA verifies** test beads (tests fail for right reasons, no impl code)
3. **Assign:** T-01 → Worker-2 (green phase — different worker), T-02 → Worker-1
4. **QA verifies** impl beads (tests pass, acceptance criteria met)
5. **Wave Gate:**
   - Integration quality gates: `uv run ruff check . && uv run pytest -v`
   - Review flywheel: correctness + security + compaction (3 agents)
   - Smoke test: health endpoint, new endpoints from this wave
6. **Checkpoint written** to <feature_dir>/sprint-state.md

### Wave 2 — Core (estimated: X tickets)
[Same structure...]

### Sprint Close
1. Quality gates (full suite)
2. docs-gen-int → writes architecture.md, api.md, cli.md, etc. to <feature_dir>/
3. docs-gen-ext → writes to docs/areas/site/features/<name>/, guides/, reference/
4. fresh-eyes → cold-read audit of entire <feature_dir> (code + plan + docs + beads)
5. land-the-plane → commit + push
6. Lifecycle bead marked complete, final checkpoint written

### Summary
- Total agents: 1 director + N workers + 1-2 QA + 3-4 standing lens reviewers
- Total waves: N
- Estimated tickets: N (N impl + N test)
- Artifacts produced: sprint-plan.md, sprint-state.md, architecture.md, api.md,
  cli.md, data-model.md, what-shipped.md, lessons.md, docs/areas/site/...
```

4. **Flag risks:** highlight any beads that look problematic (missing acceptance
   criteria, vague scope, no TDD pair, suspicious dependencies)
5. Ask: "This is what the sprint will do. Proceed with `/sprint-go` (no --dry-run) to launch, or adjust?"

**Dry-run does NOT:**
- Create teams, spawn agents, or modify beads
- Write any files (no checkpoint, no sprint-plan.md)
- Run quality gates or tests

---

## Live Mode (default — no --dry-run flag)

### Step 1 — Read Sprint Plan

- Read `tmp/sprint-exec-plan.md`
- If missing: tell user to run `/hs-sw-sprint-exec-plan` first and stop
- Present quick summary: "About to launch N workers across K waves. Confirm?"

### Step 1b — Status Line Check

- Check `.claude/settings.json` for `statusLine` config
- If missing or `tmp/sprint-status.sh` doesn't exist: warn the user
  ```
  ⚠ Sprint status bar not configured. Run /hs-sw-sprint-exec-plan to generate it,
    or you won't see live progress in the Claude Code status bar.
  ```
- If present: confirm it's active — `bash tmp/sprint-status.sh` and show the output

### Step 2 — Approval Gate

- Show: ticket count, agent count, estimated cost tier distribution
- Show: domain balance (backend/frontend/infra coverage)
- Show: TDD coverage (how many impl beads have companion test beads)
- **Wait for explicit "go" from user**
- Options: go / cancel

### Step 3 — Spawn Full Topology

The launcher does ALL spawning. Load tools first: `ToolSearch("select:Agent,SendMessage")`.

**Teams model (Claude Code ≥ v2.1.178):** `TeamCreate`/`TeamDelete` were REMOVED. Teams now form
implicitly when the launcher spawns background agents in the same session — gated behind the
experimental flag `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (set in `.claude/settings.json` `env`,
applied at launch) and best supported in the interactive TUI. There is NO `TeamCreate` call and NO
`team_name` parameter. Verify `Agent` + `SendMessage` are both present via ToolSearch; if
`SendMessage` is missing, the flag is off or the session is headless — tell the user to set the
flag and relaunch in an interactive session, and fall back only to an explicit, user-approved
solo-sequential run (never silent).

1. **Spawn the Director** as a background teammate (model: **fable** — see AGENTS.md Model Tiers):
   `Agent(subagent_type="hs-sw-sprint-director", name="director", run_in_background=true, model="fable", mode=<sprint autonomy mode>)`
   Its prompt is the FULL sprint brief:
   - Complete `tmp/sprint-exec-plan.md` content
   - **`feature_dir`** path — Director writes sprint-state.md checkpoints there
   - The roster it commands (exact teammate names spawned below)
   - Quality gates and project context from AGENTS.md/CLAUDE.md
   - Explicit statement: "You coordinate via SendMessage ONLY. You cannot and must
     not spawn agents. Your roster is fixed; if it proves insufficient, write the
     gap to sprint-state.md and message the human as last resort."
3. **Spawn workers** per the exec-plan topology — model from the lane's highest
   tier label (`tier:fable`→"fable", `tier:opus`→"opus", `tier:sonnet`→"sonnet"):
   `Agent(subagent_type="general-purpose", name="worker-N", run_in_background=true, model=<tier>, mode=<sprint autonomy mode>)`
   Worker prompt: lane assignment + "Your manager is `director`. Route ALL
   questions, blockers, and completion reports to it via SendMessage. Never
   address the human. Wait for assignments from director; do not self-start."
4. **Spawn QA agent(s)** — a SMALL FIXED POOL, never one per ticket (1 per 1-3
   workers; 2 split by domain for 4-5; model "sonnet"):
   `Agent(subagent_type="hs-sw-sprint-qa", name="qa-N", run_in_background=true, model="sonnet", mode=<sprint autonomy mode>)`
   Each QA instance works its queue of beads sequentially; the pool gives the
   parallelism. Heavyweight steps (`npm run build`, full suite runs) are
   serialized across instances by the Director.
5. **Spawn standing reviewers** (replaces Director-spawned ephemeral reviewers —
   the Director cannot spawn): one `hs-sw-sprint-bug-hunter` per lens used by the
   plan (correctness, security, compaction, + ux for frontend-heavy sprints; model "opus"):
   `Agent(subagent_type="hs-sw-sprint-bug-hunter", name="review-<lens>", run_in_background=true, model="opus", mode=<sprint autonomy mode>)`
   Reviewer prompt: "You hold the <LENS> lens for the whole sprint. Idle until
   `director` messages you a wave scope (diff range + context); review ONLY that
   scope, file beads per your definition (ALWAYS `--label=caught:review`), report
   back to director, then idle. Treat each wave as fresh — do not carry prior-wave
   conclusions into a new wave's review."
6. (Optional, plan-dependent) **Spawn peer reviewer** the same way for end-of-wave
   convergence passes (see hs-sw-peer-reviewer definition for its timing rules).

### Step 3b — Arm the Stall-Watchdog (institutionalized from jl7w3, 2026-06-24)

The Director runs as a background agent and **cannot self-wake**, so it reliably parks at
wave boundaries (jl7w3: 4 stalls, incl. a 44-min launch stall, each caught only because the
human had set up a manual `/loop` timer). Bake that timer in. Before exiting, arm a recurring
watchdog so a stall is caught in minutes, not whenever the human next checks:

`ToolSearch("select:CronCreate")`, then `CronCreate(cron="*/5 * * * *", recurring=true, prompt=<stall-check>)`
where `<stall-check>` instructs: run `bash tmp/sprint-status.sh` + `date -u`, compare to the
prior check; if there has been **NO progress for ~5+ min AND agents are idle with open
in-flight work**, diagnose the stalled bead + who owns the next action and nudge the
responsible teammate (usually the Director) via `SendMessage`. Hard rules for the watchdog:
never assign work itself; never nudge when work is genuinely in flight (a worker mid-impl, QA
mid-verify, a lens mid-review, a long e2e, or invisible gate-closeout bookkeeping — these
produce no status/file change for up to ~10 min, so widen the threshold right after a gate);
self-delete the cron (`CronDelete`) once the status bar shows all-done.

### Step 4 — Exit

- Report to user: "Team created: director + N workers + N QA + N reviewers. Director
  has the brief and manages everything via SendMessage."
- "You can walk away. Beads get the `qa-passed` label as they finish — you close them
  after review with `bd close`."
- "When the sprint is done, run `/sprint-close` to close all qa-passed beads and remove the status bar. Use `--dry-run` first to preview. If the bar shows stale data before then, run `/sprint-status-sync`."
- **Do NOT assign work yourself. Do NOT stay in the loop — the Director runs the sprint.**

## Rules

- Never proceed without explicit user approval (live mode only — dry-run needs no approval)
- The LAUNCHER spawns the entire topology as background teammates (Director CANNOT spawn —
  subagent nesting cap); the Director coordinates via SendMessage only
- Every spawn names the Director as manager and sets an explicit permission `mode` —
  this is what prevents questions and permission prompts up-leveling to the human
- Director gets the full sprint brief + the exact roster in its spawn prompt
- 1-3 workers → 1 QA agent; 4-5 workers → 2 QA agents (split by domain). QA instances
  verify different beads concurrently; Director serializes heavyweight steps (builds)
- Worker count = the exec-plan's TRUE PARALLEL WIDTH (see exec-plan Step 7); hard
  ceiling 5; QA/reviewers do not count toward it
- Wave gates are hard — no Wave N+1 work until gate passes (QA + quality gates + review flywheel + smoke test)
- Review flywheel: standing reviewer teammates, one per lens (correctness, security,
  compaction, + UX for frontend waves), pre-spawned by the launcher; Director messages
  them each wave's scope; they file beads with `--label=caught:review` — MANDATORY,
  this label feeds the escape-rate metric
- Every fix-bead filed after a ticket was qa-passed MUST carry a `caught:review|manual|pr`
  label, whoever files it (Director included) — unlabeled repair work is invisible to
  the metric and was the GH#360 retro's accounting gap
- Sprint close includes docs generation: `/hs-sw-docs-gen-int` (internal) + `/hs-sw-docs-gen-ext` (external to `docs/areas/site/`)
- Workers are general-purpose (full tool access)
- NO Tasks — beads (`bd`) is the only tracking system
- Launcher exits immediately after spawning the topology
- Requires Claude Code agent-teams enabled in the LAUNCHER session: the experimental flag
  `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (`.claude/settings.json` `env`, applied at launch)
  AND an interactive session. Verify `Agent` + `SendMessage` are available via ToolSearch
  BEFORE spawning; if `SendMessage` is absent (flag off or headless), tell the user to set the
  flag + relaunch interactively, and fall back only to an explicit, user-approved
  solo-sequential run (never silent)
