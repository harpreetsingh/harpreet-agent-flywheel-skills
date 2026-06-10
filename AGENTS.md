# AGENTS.md — harpreet-agent-flywheel-skills

> Instructions for AI agents working on this repository. Read this FIRST before
> touching any files.

## Project Overview

**harpreet-agent-flywheel-skills** — A collection of reusable Claude Code skills
and autonomous agent definitions for iterative software development. The flywheel
covers the full development lifecycle: shaping → planning → decomposition →
sprint execution → human review → docs → ship.

### Repo Structure

```
skills/          Claude Code skill definitions (SKILL.md per skill)
agents/          Autonomous agent definitions (.md per agent)
templates/       Reusable templates (PLAN-TEMPLATE.md, AGENTS-TEMPLATE.md)
playbooks/       End-to-end workflow guides
docs/            Feature plans, decisions, and supporting docs
install.sh       Symlinks skills + agents into ~/.claude/
```

---

## Issue Tracking

All work is tracked in the **GitHub Issues project**:

- **Project board:** https://github.com/users/harpreetsingh/projects/3/views/1
- **Repository issues:** https://github.com/harpreetsingh/harpreet-agent-flywheel-skills/issues

### Status columns

| Column | Meaning |
|--------|---------|
| Backlog | Filed, not yet prioritized |
| Ready | Prioritized, unblocked, available to pick up |
| In progress | Actively being worked |
| In review | PR open, awaiting review |
| Done | Merged |

### Priority labels

| Label | Meaning |
|-------|---------|
| P0 | Critical — blocks other work or causes incorrect agent behavior |
| P1 | High — meaningful improvement to skill/agent quality |
| P2 | Medium — polish, documentation, nice-to-have |

### Filing issues

```bash
gh issue create --title "..." --body "..." --label "P1"
```

### Linking to the project

After creating an issue, add it to the project:

```bash
gh issue edit <number> --add-project "agent-flywheel"
```

Or set status directly via GraphQL if needed.

---

## Model Tiers (sprint + agent work)

Three tiers, assigned per bead via `bd update <id> --add-label tier:<tier>`
(policy set 2026-06-10; Fable is the newest, most capable model):

| Tier | Use for | Examples |
|---|---|---|
| `tier:fable` | **Heavy** — architectural decisions, complex multi-file refactors, high fan-out (blocks 3+), protocol/auth/security-critical design, subtle debugging | middleware auth semantics, execution-engine changes, cross-cutting migrations |
| `tier:opus` | **Standard** — features, API endpoints, UI components, test writing, contract tests, non-trivial fixes | new endpoint + proxy + dropdown, red-phase test suites |
| `tier:sonnet` | **Trivial** — mechanical/boilerplate, config, docs, label/copy changes, straightforward verification checklists | relabel a field, regenerate an index, checklist QA |

Role defaults: **Director = fable** (coordination judgment is the highest-leverage
spend — escape rates are driven by planning quality), **lens reviewers = opus**,
**QA = sonnet** (checklist-driven), workers = their bead's tier. Do NOT use haiku.

## QA Parallelization Policy

QA is a SMALL FIXED POOL, parallelized by logical group — **never one agent per
ticket**. 1–3 workers → 1 QA; 4–5 workers → 2 QA split by domain
(backend/frontend). Each QA instance processes its queue of beads sequentially;
parallelism comes from the pool, and heavyweight steps (`npm run build`, full
suite runs) are serialized across instances by the Director.

## Work Execution

### Starting work

1. Find a **Ready** issue on the project board
2. Move it to **In progress**
3. Create a branch: `git checkout -b fix/<issue-slug>` or `feature/<issue-slug>`

### During work

- Skills live at `skills/<name>/SKILL.md` — one file per skill
- Agents live at `agents/<name>.md` — one file per agent
- Test a skill by reading it cold and asking: "could an agent execute this with no other context?"
- Check FLYWHEEL.md is consistent if you change a skill's inputs/outputs

### Completing work

1. Self-review: re-read every file you changed
2. Verify install.sh still works if you added/renamed files
3. Open a PR — move issue to **In review**
4. Human merges — moves issue to **Done**

---

## Git Safety Protocol

### Forbidden without explicit approval

- `git reset --hard`
- `git clean -fd`
- `git push --force`
- `rm -rf` on any directory

### Commit discipline

- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing — that leaves work stranded locally
- Small logical commits: one skill change per commit, not a bulk dump

### Branch strategy

```
feature/<slug> or fix/<slug> → main (PR) → merged
```

Never commit directly to `main`.

---

## Code Editing Rules

### Surgical edits

Never rewrite a skill or agent file wholesale. Before modifying any file:

1. Read the entire file first
2. Identify the exact section to change
3. Change ONLY that — preserve all other instructions, protocols, and rules

If you find yourself replacing more than ~30% of a file, stop and reconsider.

### No file sprawl

- No `_v2`, `_improved`, `_new` variants — revise in place
- New files are rare — incredibly high bar
- If renaming a skill, update `install.sh` and `README.md` in the same commit

### Consistency

- New skills must follow the naming convention: `hs-<category>-<name>/SKILL.md`
- Categories: `sw` (software), `mkt` (marketing), `cc` (Claude Code)
- Update `README.md` skills table when adding or removing a skill
- Update `FLYWHEEL.md` if the skill connects to the development lifecycle

---

## Quality Check (before every PR)

This repo has no build system. The quality check is manual:

1. **Read your changes cold** — pretend you're an agent seeing this for the first time
2. **Check cross-references** — if you updated a skill, does FLYWHEEL.md still describe it correctly?
3. **Check install.sh** — does it still reference the right paths?
4. **Check README.md** — is the skills table up to date?
