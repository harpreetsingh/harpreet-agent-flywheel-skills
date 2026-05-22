---
name: hs-sw-project-init
description: Bootstrap a new project with AGENTS.md, beads, quality gates, and standard structure
argument-hint: [project-name]
---

# /project-init — Bootstrap a New Project

Set up a new project (or an existing project missing agent infrastructure) with
the standard development workflow: AGENTS.md, beads, quality gates, decision
docs, and gitignore.

## Process

1. **Detect the project.** Look at the current directory:
   - Is there a `package.json`? → Node/TypeScript project
   - Is there a `pyproject.toml`? → Python project
   - Is there a `Cargo.toml`? → Rust project
   - Is there a `go.mod`? → Go project
   - Multiple? → Monorepo (backend + frontend)
   - None? → Ask the user what stack they're using

2. **Check what already exists.** Don't overwrite existing files:
   - `AGENTS.md` — skip if exists, offer to merge
   - `CLAUDE.md` — skip if exists
   - `.beads/` — skip if exists
   - `.gitignore` — merge entries, don't overwrite
   - `docs/decisions/` — create if missing

3. **Create AGENTS.md** from the template at `templates/AGENTS-TEMPLATE.md`
   in the claude-workflow-skills repo. Customize:
   - Fill in project name from `$ARGUMENTS` or directory name
   - Fill in tech stack from detection in step 1
   - Fill in project structure from `ls`
   - Customize quality gate commands for the detected stack:
     - **Python:** `uv run ruff check .` → `uv run ruff format --check .` → `uv run pytest -v`
     - **TypeScript/JS:** `npm run lint` → `npx tsc --noEmit` → `npm run build`
     - **Rust:** `cargo clippy -- -D warnings` → `cargo test`
     - **Go:** `go vet ./...` → `go test ./...`
   - Remove sections that don't apply (e.g., no Frontend section for a pure backend project)

4. **Create CLAUDE.md** if it doesn't exist. Minimal version:
   ```markdown
   # CLAUDE.md

   ## Project Overview
   **[Project Name]** — [description from user or inferred]

   See [AGENTS.md](./AGENTS.md) for all development instructions.

   ## Work Execution Contract
   See AGENTS.md — agents MUST follow the workflow defined there.

   ## Session Completion
   See AGENTS.md "Landing the Plane" section. Work is NOT done until `git push` succeeds.
   ```

5. **Initialize beads** if `.beads/` doesn't exist:
   ```bash
   bd init
   ```

6. **Create docs structure:**
   ```
   docs/
   ├── decisions/       # Architectural decision records
   └── learnings.md     # Session learnings (empty template)
   ```

7. **Update .gitignore** — merge these entries if not already present:
   ```
   # Build artifacts
   *.tsbuildinfo
   .next/
   __pycache__/
   node_modules/
   dist/
   build/

   # Environment
   .env
   .env.local
   .env.*.local

   # Local state
   .DS_Store
   *.log
   *.sqlite3
   .beads/.local_version
   .beads/daemon-error

   # IDE
   .idea/
   .vscode/
   *.swp
   ```

8. **Summary.** Report what was created and what was skipped. Remind the user to:
   - Fill in the project overview in AGENTS.md
   - Create a `PLAN.md` using `/plan-draft` or the template
   - Commit the new files

## Rules

- NEVER overwrite existing files without asking.
- Detect the stack — don't ask the user what they already have in their repo.
- Keep AGENTS.md lean. Project-specific details go in CLAUDE.md, not AGENTS.md.
- The AGENTS.md template is the starting point, not gospel. Remove irrelevant
  sections rather than leaving empty placeholders.
