# claude-workflow-skills

Reusable [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills and agents for iterative software development.

## Scaling & quality layer (v1.1)

Beyond v1.0's TDD + QA + wave-gating, the flywheel now has the mechanisms to scale agent count *safely* — built around climbing [Steve Yegge's 8 levels of AI-assisted development](https://www.augmentcode.com/guides/steve-yegge-8-levels-ai-assisted-development) past Level 6. Correctness-first: prevent rework at the source, measure it, and only scale when the data says it's safe.

| Capability | What it does | Where |
|---|---|---|
| **Escape-rate metric** | Durable cross-project rework signal (`caught:review/manual/pr` + `[QA-FAIL]`) → `~/.claude/flywheel/sprint-metrics.jsonl`. The 20% scaling gate. | sprint-qa, sprint-close, `/flywheel-metrics` |
| **Learning loop** | Failure patterns → proposed Phase 0 / AGENTS.md fixes that drive the next sprint's escape rate down | `/sprint-retrospective` |
| **Right-sizing** | Mechanical bead-size limits (one bead/one layer/one deliverable) — the #1 rework predictor | beads-create, beads-review |
| **Interface contracts** | Producer & consumer beads carry byte-identical `## Contract` blocks — defeats silent spec drift | beads-create, beads-review |
| **Static collision avoidance** | Beads declare `## Files`; review builds a file-overlap graph; Director schedules collision-free by construction | beads-create, beads-review, exec-plan, director |
| **Review at scale** | Lenses shard by diff volume; findings dedup into one ranked Review Digest (no review collapse) | director |
| **Per-ticket recovery** | Append-only `sprint-log.md` event log; recovery replays the tail (loss window: one event, not one wave) | director |

See [`FLYWHEEL.md`](FLYWHEEL.md) for the full lifecycle map. agent_mail (runtime file leases) is the planned next layer — its leases consume the `## Files` sets already declared here.

## Naming Convention

All skills use the `hs-` namespace with a category prefix for discoverability:

| Prefix | Category | Description |
|--------|----------|-------------|
| `hs-sw-` | Software | SDLC skills — planning, quality, shipping, docs |
| `hs-mkt-` | Marketing | Content capture, blog drafting, indexing |
| `hs-cc-` | Claude Code | Session management — stash and hydrate |

Type `/hs-` to see all skills. Type `/hs-sw-` to narrow to software. Type `/hs-mkt-` for marketing.

## What's included

### Templates

| File | Purpose |
|------|---------|
| `templates/PLAN-TEMPLATE.md` | Canonical plan structure — scales from feature to architecture |
| `templates/AGENTS-TEMPLATE.md` | Universal AGENTS.md — git safety, beads workflow, quality gates, code discipline |

### Playbooks

| File | Purpose |
|------|---------|
| `playbooks/iterative-development.md` | Full workflow from decision docs → shipped code |

### Skills (slash commands)

#### Software (`hs-sw-`)

Ordered by flywheel phase (bootstrap → shape → plan → decompose → sprint → close → support).

| Command | Purpose |
|---------|---------|
| `/hs-sw-project-init [name]` | Bootstrap a new project with AGENTS.md, beads, quality gates, docs structure |
| `/hs-sw-flywheel [phase]` | Render the full development flywheel as a visual map; write FLYWHEEL.md; zoom into any phase |
| `/hs-sw-shape [feature]` | Run a Shape Up shaping interview (5 rounds) and produce a pitch.md |
| `/hs-sw-plan-draft [source-dir]` | Synthesize decision docs into a structured PLAN v1 |
| `/hs-sw-plan-review [file]` | Iteratively review and improve a markdown plan |
| `/hs-sw-beads-create [file] [--labels a,b]` | Create epics/tasks/subtasks with deps, `## Files`, `## Contract`, and `## Steps` from a plan |
| `/hs-sw-beads-review` | Optimize beads: self-sufficiency, **right-sizing**, **interface contracts**, **file-overlap graph** |
| `/hs-sw-beads-label <epic-id> --labels a,b` | Add labels to an epic and all its tickets (or an explicit bead-id list) |
| `/hs-sw-sprint-exec-plan` | Analyze beads into waves + cost tiers + team topology; worker count gated by escape rate & true parallel width |
| `/hs-sw-sprint-go` | Launch multi-agent sprint from execution plan — spawn director + workers |
| `/hs-sw-sprint-recover [dir]` | Triage a failed/incomplete sprint — fix beads state, prepare for re-sprint |
| `/hs-sw-sprint-status-sync` | Refresh the sprint status bar when displayed status diverges from beads reality |
| `/hs-sw-sprint-status-clear` | Remove the sprint status bar from Claude Code |
| `/hs-sw-sprint-close [--dry-run]` | Close a finished sprint: close qa-passed beads, log escape rate, remove status bar, clean tmp |
| `/hs-sw-sprint-retrospective [dir]` | Turn a sprint's failure data into patterns + proposed Phase 0/AGENTS.md fixes — drives escape rate down |
| `/hs-sw-flywheel-metrics` | Show the rework/escape-rate trend across sprints — the 20% scaling gate |
| `/hs-sw-test-coverage [dir]` | Find test gaps and create beads for missing tests |
| `/hs-sw-ux-polish [path]` | Deep UI/UX scrutiny targeting Stripe-level quality |
| `/hs-sw-fresh-eyes [path]` | Cold-read audit of code, plans, docs, or beads for bugs and gaps |
| `/hs-sw-ci-preflight` | Run all tests/linters locally matching exact GitHub Actions CI commands before push |
| `/hs-sw-land-the-plane [hint]` | Quality gates + logical commits + push |
| `/hs-sw-docs-gen-int [version]` | Synthesize internal engineering docs for a release from all artifacts |
| `/hs-sw-docs-gen-ext [source-dir]` | Extract external-facing docs from decision docs and PRDs |
| `/hs-sw-repeat [N] /<skill>` | Run any review skill in rounds until convergence |
| `/hs-sw-gh-issue [repo]` | File a GitHub issue with title, body, labels, and native type via GraphQL |
| `/hs-sw-gh-beads-link <issue> [beads-id]` | Bidirectionally link a beads ticket ↔ GitHub issue (sets external-ref, project Beads field, backlink comment) |

#### Marketing (`hs-mkt-`)

| Command | Purpose |
|---------|---------|
| `/hs-mkt-capture [description]` | Quickly capture an interesting insight mid-session for future content |
| `/hs-mkt-blog-draft [files-or-topic]` | Draft a v1 blog post from captured content, move sources to done |
| `/hs-mkt-content-index` | Rebuild the content index grouped by project and topic |

#### Claude Code (`hs-cc-`)

| Command | Purpose |
|---------|---------|
| `/hs-cc-stash [focus]` | Save session context before compaction or ending a session |
| `/hs-cc-hydrate` | Restore session context from the most recent stash |

### Agents (autonomous subagents)

| Agent | Purpose |
|-------|---------|
| `hs-sw-bug-hunter` | Strategically explore code from entry points, trace execution flows, find and fix bugs |
| `hs-sw-peer-reviewer` | Review code from fellow agents/humans across recent commits |
| `hs-sw-sprint-director` | Autonomous sprint director — wave management, task assignment, role switching |

## Install

```bash
git clone https://github.com/harpreetsingh/claude-workflow-skills.git
cd claude-workflow-skills
chmod +x install.sh uninstall.sh
./install.sh
```

This creates symlinks from `~/.claude/skills/` and `~/.claude/agents/` into the repo. Updates are just `git pull`.

## Update

```bash
cd claude-workflow-skills
git pull
# Symlinks already point here — changes take effect immediately
```

## Uninstall

```bash
cd claude-workflow-skills
./uninstall.sh
```

## Usage

In any Claude Code session:

```
/hs-sw-plan-draft docs/decisions/    # Synthesize docs into PLAN v1
/hs-sw-plan-review PLAN.md           # One round of plan improvement (repeat ×4-5)
/hs-sw-fresh-eyes                    # Review recent changes for bugs
/hs-sw-beads-create PLAN.md          # Turn plan into trackable beads
/hs-sw-beads-review                  # Optimize bead structure
/hs-sw-test-coverage backend/        # Find test gaps
/hs-sw-ux-polish                     # Full UX audit
/hs-sw-land-the-plane "feat: add X"  # Quality gates + commit + push
/hs-mkt-capture "interesting insight" # Capture content mid-session
/hs-mkt-blog-draft captures/file.md  # Draft a blog from captures
/hs-cc-stash "working on auth flow"  # Save context before compaction
/hs-cc-hydrate                       # Restore context in new session
```

Agents are invoked automatically by Claude Code when it recognizes a matching task, or you can reference them directly.

## Workflow

A typical development cycle:

1. **Bootstrap** — `/hs-sw-project-init my-app`
2. **Draft** — `/hs-sw-plan-draft docs/decisions/` or write `PLAN.md` from `templates/PLAN-TEMPLATE.md`
3. **Refine** — `/hs-sw-plan-review PLAN.md` × 4-5 rounds until convergence
4. **Decompose** — `/hs-sw-beads-create PLAN.md`, then `/hs-sw-beads-review`
4.5. **Sprint** (optional) — `/hs-sw-sprint-exec-plan` then `/hs-sw-sprint-go`
5. **Implement** — Work through beads (or let the sprint director handle it)
6. **Test** — `/hs-sw-test-coverage`
7. **Polish** — `/hs-sw-ux-polish`
8. **Verify** — `/hs-sw-fresh-eyes`
9. **Ship** — `/hs-sw-land-the-plane`
10. **Pause** — `/hs-cc-stash` (before compaction or ending session)
11. **Resume** — `/hs-cc-hydrate` (in new session)
12. **Capture** — `/hs-mkt-capture "interesting insight"` (mid-session, anytime)
13. **Blog** — `/hs-mkt-blog-draft captures/file.md` (when ready to write)
14. **Release docs** — `/hs-sw-docs-gen-int v0.17b` (at release close)
15. **External docs** — `/hs-sw-docs-gen-ext docs/prds/versions/v0.17/` (at release close)

See `playbooks/iterative-development.md` for the full workflow guide.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [beads](https://github.com/steveyegge/beads) (`bd` CLI) for task tracking skills
- `git` for `/hs-sw-land-the-plane` and `/hs-sw-fresh-eyes`
- Claude Code team features for `/hs-sw-sprint-go`

## License

MIT
