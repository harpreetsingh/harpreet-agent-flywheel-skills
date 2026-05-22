---
name: hs-sw-ci-preflight
description: Run all tests and linters locally matching exact GitHub Actions CI commands — catch failures before push
argument-hint: [--fix]
---

# /ci-preflight — CI Parity Check

Run every check that GitHub Actions will run, locally, before you push.
The goal is zero surprises on CI. If this passes, CI passes.

Pass `--fix` to auto-fix lint/format issues where possible.

## Process

### 1. Discover what CI actually runs

```bash
ls .github/workflows/
```

Read every `.github/workflows/*.yml` file. For each workflow:
- Identify jobs that trigger on `push` to main/master OR `pull_request`
- Extract the exact `run:` commands in those jobs, in order
- Note any `working-directory:` overrides
- Note any required environment variables or secrets (flag them — you can't replicate secrets locally)

If no `.github/workflows/` exists: fall back to stack detection (see step 2b).

**Output a CI map** before running anything:
```
CI jobs found: ci.yml → test, lint, typecheck
Commands to replicate:
  [test]       cd backend && uv run pytest -v
  [lint]       cd backend && uv run ruff check .
  [format]     cd backend && uv run ruff format --check .
  [typecheck]  cd frontend && npx tsc --noEmit
  [lint-fe]    cd frontend && npm run lint
```

### 2. Run checks in CI order

Run each command exactly as CI would. For each check:
- Print the command before running it
- Capture stdout + stderr
- Record pass ✓ or fail ✗ with exit code

**2b. Fallback stack detection** (if no CI config found):

| Indicator | Checks to run |
|-----------|---------------|
| `backend/pyproject.toml` or `pyproject.toml` | `uv run ruff check .` → `uv run ruff format --check .` → `uv run pytest -v` |
| `frontend/package.json` or `package.json` | `npm run lint` → `npx tsc --noEmit` → `npm run build` |
| `Cargo.toml` | `cargo clippy -- -D warnings` → `cargo test` |
| `go.mod` | `go vet ./...` → `go test ./...` |

Run ALL applicable stacks — monorepos have multiple.

### 3. Handle --fix

If `--fix` was passed:
- Re-run failed lint/format commands with their auto-fix equivalent:
  - `ruff check . --fix --unsafe-fixes`
  - `ruff format .`
  - `eslint --fix`
  - `cargo fmt`
- Re-run the check after fixing to confirm it now passes
- Report what was changed

### 4. Report results

Print a final summary table:

```
CI Preflight Results
────────────────────────────────────────────
  ✓  backend/tests         pytest -v                    (47 passed)
  ✓  backend/lint          ruff check .                 (clean)
  ✓  backend/format        ruff format --check .        (clean)
  ✗  frontend/typecheck    npx tsc --noEmit             (3 errors)
  ✗  frontend/lint         npm run lint                 (2 errors)
────────────────────────────────────────────
FAILED — 2 checks failed. Fix before pushing.
```

If all pass:
```
────────────────────────────────────────────
PASSED — all 5 checks clean. Safe to push.
────────────────────────────────────────────
```

For each failure: print the **exact error output** — file, line number, rule name.
Do not summarize errors. The full output is what the developer needs.

## Rules

- **Never modify code** unless `--fix` was passed. This is a read-only check by default.
- **Run all checks**, not just the ones for changed files. CI runs everything.
- **Exit non-zero** (indicate failure) if any check fails — callers like `/land` use this to decide whether to proceed.
- If CI config references secrets (`${{ secrets.* }}`): skip that step, note it was skipped, do not fail on it.
- If a command is not installed (e.g., `uv` missing): report it as a setup error, not a check failure. Tell the user how to install it.
- Do NOT run `npm install` or `uv sync` automatically — if dependencies are missing, report it and stop.
