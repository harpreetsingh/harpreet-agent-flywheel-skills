# AGENTS.md

> Instructions for AI agents working on this project. Read this FIRST before
> touching any code.

## Project Overview

**[Project Name]** — [One-sentence description of what this project does.]

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | [e.g., Python 3.14, TypeScript 5.x] |
| Backend | [e.g., FastAPI, Express, Rails] |
| Frontend | [e.g., Next.js, SvelteKit] |
| Database | [e.g., PostgreSQL, SQLite] |
| Deployment | [e.g., Railway, Vercel, AWS] |
| Package Manager | [e.g., uv, npm, pnpm, bun] |

### Project Structure

```
[fill in with top-level directory layout]
```

### Component Map (Frontend Projects)

> Skip this section for backend-only or CLI projects.

Agents working on frontend tickets must know which file owns which UI region.
Without this map they guess — and incorrect placement, duplicate components,
and edits to the wrong file are the result.

**App shell / root layout:**

```
[fill in render tree from root layout — e.g.:
RootLayout (app/layout.tsx)
├── AuthGate (components/auth/...)
└── AppShell (components/layout/...)
    ├── Sidebar (components/layout/sidebar)
    └── Main content → page.tsx per route]
```

**Route → file mapping:**

| Route | Page file | Key components rendered |
|-------|-----------|------------------------|
| `/` | `app/page.tsx` | [fill in] |
| [add all routes] | | |

**Component directory guide:**

| Directory | Purpose |
|-----------|---------|
| `components/ui/` | Shared primitives (shadcn/ui or equivalent) — never modify directly |
| `components/layout/` | App shell, nav, sidebar |
| [add domain directories] | |

**Rules derived from this map:**
- Before writing any frontend bead, look up the target route here to get the authoritative file
- Never modify files in `components/ui/` unless the request is explicitly about the design system
- When two files have similar names, this map is the tiebreaker

---

## Work Execution Contract

**Every agent MUST follow this workflow.**

### Starting Work

1. `bd ready` — find available work (no blockers)
2. `bd show <id>` — read full context before touching code
3. `bd update <id> --status in_progress` — claim it immediately

### During Work

- `bd comments add <id> "description"` — log progress and decisions
- Keep status as `in_progress` until ALL acceptance criteria verified
- If blocked, create a new issue for the blocker: `bd create --title="..." --type=bug`

### Completing Work

1. **Test the implementation** — "code written" ≠ "done"
2. Add a final comment summarizing what was done
3. `bd close <id>`
4. `bd sync --flush-only` — export to JSONL
5. Commit with `.beads/issues.jsonl` included

### Rules

- **Always use `bd` CLI** — NEVER directly read or write `.beads/issues.jsonl`
- **Always sync before committing** — `bd sync --flush-only`
- **Never mark done without testing**
- **Update status in real-time** — status should reflect actual state

---

## Git Safety Protocol

### Forbidden Without Explicit Approval

- `git reset --hard`
- `git clean -fd`
- `git push --force`
- `rm -rf` on any directory
- Any irreversible file deletion

### For Any Destructive Operation

1. State the exact command you will run
2. List exactly what will be affected
3. Wait for explicit approval
4. Execute only the approved command

### Commit Discipline

- Work is NOT complete until `git push` succeeds
- NEVER stop before pushing — that leaves work stranded locally
- If push fails, resolve and retry until it succeeds
- Never say "ready to push when you are" — YOU must push

### Pre-Commit Artifact Rejection

**NEVER commit these files.** If they appear in `git status`, add to `.gitignore`
and unstage before committing:

- `*.tsbuildinfo` — TypeScript build cache
- `*.log` — log files
- `.env*` (except `.env.example`) — secrets
- `__pycache__/` — Python bytecode
- `node_modules/` — JS dependencies
- `.next/` — Next.js build output
- `.DS_Store` — macOS artifacts
- `*.sqlite3` — local databases
- `.beads/.local_version` — beads local state
- `.beads/daemon-error` — beads runtime log

---

## Quality Gates

**Run before every commit. Every failed push costs real money (CI, deployments).**

<!-- Customize these for your stack. Examples below. -->

### Backend

```bash
# Python (uv + ruff + pytest)
cd backend
uv run ruff check .            # MUST pass
uv run ruff format --check .   # MUST pass
uv run pytest -v               # MUST pass
```

### Frontend

```bash
# TypeScript (npm + eslint + tsc)
cd frontend
npm run lint                   # MUST pass
npx tsc --noEmit              # MUST pass
npm run build                  # MUST pass
```

---

## CLI Design Principles

Every user-facing feature should be accessible via CLI, not just UI/API.

- Every command supports `--json` for machine/agent consumption
- Human-friendly output by default, structured JSON when piped or flagged
- CLI is a first-class interface, not an afterthought

---

## Verification Entry Points

Manual testing surfaces for sprint verification. `/hs-sw-sprint-exec-plan`
creates tickets for any missing entry points listed here.

<!-- Customize for your project. Examples: -->
<!-- - Demo page: `frontend/app/demo/page.tsx` — exercises routes and endpoints -->
<!-- - Docs endpoint: `backend/docs/` — interactive API documentation -->
<!-- - Health check: `GET /api/health` — smoke test for dependencies -->

---

## Session Completion ("Landing the Plane")

**When ending a work session**, complete ALL steps. Work is NOT done until
`git push` succeeds.

1. **File issues** for remaining work — `bd create --title="..." --type=task`
2. **Run quality gates** (if code changed)
3. **Update issue status** — close finished work, update in-progress items
4. **Sync and push:**
   ```bash
   bd sync --flush-only
   git pull --rebase
   git push
   git status  # MUST show "up to date with origin"
   ```
5. **Hand off** — provide context for next session

---

## Code Editing Rules

### No Automated Refactoring

- NEVER run scripts that process/change code files (no codemods, no sed/regex)
- For many simple changes: use subagents in parallel, each making manual edits
- For subtle/complex changes: do methodically, file-by-file, with careful reasoning

### No File Sprawl

- No V2 suffixes, no `_improved`, no `_unified`, no `_new` files
- Revise in place: migrate callers and remove old code in the same commit
- New files are RARE — incredibly high bar. Prefer editing existing files.
- No backwards-compatibility shims — fix callers directly

### Surgical Component Edits (Frontend)

Never rewrite a component file wholesale. Before modifying any UI component:

1. Read the entire file first
2. Identify the exact function, hook, or JSX block that needs to change
3. Change ONLY that — preserve all other logic, props, component states, and event handlers

A correct result via full-file rewrite is still a failure. The diff should touch
only the lines the request requires. If you find yourself replacing more than ~30%
of a component file, stop — you are almost certainly doing more than was asked.

Component states that must be preserved unless the request explicitly changes them:
loading, empty, error, data (populated), partial (some data missing). All five.

### Third-Party Libraries

- NEVER guess at APIs — search online for current documentation first
- Verify the library version in use before writing code against it
- Understand current best practices for the library

---

## Design Decisions

When you make or encounter an architectural decision:

1. Write it to `docs/decisions/` with rationale and alternatives considered
2. Include: context, decision, consequences, alternatives rejected and why

Format: `docs/decisions/YYYY-MM-DD-short-name.md`

---

## Plan & Playbook

This project follows the iterative development playbook:

1. **Plan** — `PLAN.md` using the canonical template (Goals/Non-Goals/Context/Deliverables)
2. **Review** — `/hs-sw-plan-review PLAN.md` × 4-5 rounds until convergence
3. **Decompose** — `/hs-sw-beads-create PLAN.md` then `/hs-sw-beads-review`
4. **Implement** — work through beads (`bd ready` → implement → `bd close`)
5. **Ship** — `/hs-sw-land-the-plane` (quality gates + commit + push)

See the [iterative development playbook](https://github.com/harpreetsingh/claude-workflow-skills/blob/main/playbooks/iterative-development.md)
for the full workflow.

---

## Session Learnings

Maintain a learnings file at `docs/learnings.md`. Write entries when you discover:

- Surprising insights or patterns that saved time
- Debugging root causes (symptoms → investigation → fix)
- What was harder/easier than expected
- Process improvements and workflow wins
- Alternatives considered and tradeoffs

Format: date, category tag, one-line summary, detailed paragraph with concrete
specifics (not vague observations).
