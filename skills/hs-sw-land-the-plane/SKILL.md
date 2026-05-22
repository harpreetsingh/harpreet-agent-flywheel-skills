---
name: hs-sw-land-the-plane
description: Run quality gates, commit changes in logical groups, sync beads, and push to remote
argument-hint: [commit-message-hint]
---

# /land — Land the Plane

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → ★CLOSE  │
│ ★ YOU ARE HERE: Final step. Quality gates → commit → push.              │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Run the full pre-push checklist, commit, and push. Work is NOT done until
`git push` succeeds.

## Process

1. **Check what changed**
   ```
   git status
   git diff --stat
   ```
   If there are no changes to commit, say so and stop.

2. **Artifact check** — Scan for files that should never be committed:
   `*.tsbuildinfo`, `.beads/.local_version`, `*.log`,
   `.next/`, `node_modules/`, `__pycache__/`, `.env*` (except `.env.example`),
   `.DS_Store`, `*.sqlite3`
   If found: add to `.gitignore`, unstage, and warn.

3. **CI preflight** — Run `/hs-sw-ci-preflight` to execute all tests and linters
   matching the exact commands GitHub Actions will run. This is a hard gate:
   if preflight fails, STOP and report the failures. Do not proceed to commit.
   Pass `--fix` if you want lint/format issues auto-corrected first.

4. **Beads preflight** (if `.beads/` directory exists)
   ```
   bd preflight --check
   ```
   This runs automated checks: tests run locally, lint errors, formatting,
   version mismatches. If preflight fails, report the issues. Fix what you
   can (lint/format auto-fix), warn on the rest.

   Note: beads state lives on Railway Dolt (remote). Every `bd` write lands
   immediately — no local export or sync step needed.

4.5 **Docs graduation** — Check whether any `docs/projects/features/<name>/` directory
   appears in the changeset (from `git status`). If so, this is a feature close:
   ```
   git mv docs/projects/features/<name>/ docs/resources/features/<name>/
   ```
   This is its own commit: `"docs: graduate <name> from projects to resources"`.
   Commit it before proceeding to the remaining changes.

5. **Commit** — Create a series of logically connected commits, NOT one giant
   commit. Analyze all changed files and group them by coherent change:
   - Feature + its tests = one commit
   - Refactor = one commit
   - Config/infra changes = one commit
   - Docs/beads = one commit
   Each commit message should be detailed: subject line (what), body (why and
   what's in this group). If `$ARGUMENTS` provided, use it as the overall theme.
   Do NOT edit any code at this stage. Do NOT commit ephemeral files.

6. **Push + PR**

   First, check the current branch:
   ```
   git branch --show-current
   ```

   **If on a protected branch** (`main`, `production`):
   STOP. Do NOT push directly. Tell the user:
   "You're on `<branch>` which is protected. Create a feature branch first:
   `git checkout -b feature/<name>`, then re-run `/land`."

   **If on a feature/fix branch:**
   ```
   git pull --rebase origin <branch>
   ```
   If rebase conflicts occur, STOP and report them — do not auto-resolve.
   ```
   git push -u origin <branch>
   ```
   Then create or find the PR:
   ```
   # Check for existing PR first
   gh pr view --json url 2>/dev/null || gh pr create --base main --title "<summary>" --body "<description>"
   ```
   Report the PR URL to the user.

   **If `gh` is not available or PR creation fails:** push succeeds, tell the
   user to create the PR manually. The push is the critical path, PR is best-effort.

   **Branch strategy:** `feature/<name>` or `fix/<name>` → PR → `main` → `production`.
   No `development` branch.

7. **Monitor CI** — After the PR is open, watch GitHub Actions:
   ```bash
   gh pr checks --watch
   ```
   Wait for all checks to complete. Report final status:
   - All green → done. Report PR URL and "CI passed."
   - Any red → report which jobs failed with their log URLs:
     ```bash
     gh run view <run-id> --log-failed
     ```
   Do not exit until CI completes or the user interrupts. CI failures after a
   clean preflight are usually environment or secrets issues — flag that distinction.

## Rules

- If quality gates fail, STOP. Report the failures. Do not push broken code.
- **Never push directly to protected branches** (`main`, `production`).
  Always go through a PR, even for urgent hotfixes.
- If push fails, resolve and retry until it succeeds.
- Don't skip checks. Don't guess. Always verify.
- Every push costs real money (CI, deployments). Treat it accordingly.
