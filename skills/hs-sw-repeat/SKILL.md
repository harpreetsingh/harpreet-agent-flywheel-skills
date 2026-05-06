---
name: hs-sw-repeat
description: Run any review skill in rounds until convergence — stops manual re-invocation of /plan-review, /fresh-eyes, /beads-review, etc.
argument-hint: [N] /<skill> [skill-args]
---

# /repeat — Iterative Skill Runner

Run a skill multiple times until it converges (finds nothing left to fix) or
hits a round cap.

## Usage

```
/repeat /plan-review PLAN.md             → run until convergence
/repeat 3 /fresh-eyes src/               → exactly 3 rounds
/repeat /beads-review                    → run until convergence
/repeat 5 /ux-polish src/components/     → exactly 5 rounds
```

## Parse Arguments

1. If `$ARGUMENTS` starts with a number → that's the **round cap** (extract it)
2. Next token starting with `/` → that's the **skill name** (extract it)
3. Everything remaining → passed as **skill arguments**
4. If no round cap specified → default cap is **5 rounds**

## Process

### Before starting

Report:
```
Repeat: running /<skill> <args> for up to N rounds.
```

### Each round

1. **Announce:** `Round N/cap:`
2. **Invoke the skill** with its arguments (run it as you normally would)
3. **Collect results.** The skill will produce output — typically an issues table
   if it follows the Output Protocol, or a report/fixes if it doesn't.
4. **Score the round:**
   - Count issues found (High / Med / Low)
   - Count fixes applied
   - Note if the skill itself signaled convergence (e.g., "this plan has
     likely converged", or "clean — no issues found")

### After each round — convergence check

A round is **converged** if ANY of these are true:
- The skill found **0 issues** (clean pass)
- The skill explicitly signaled convergence in its output
- All issues found were **Low** severity only (no High or Med remaining)
- The issue count **did not decrease** from the previous round (plateau —
  the skill is finding the same things repeatedly)

If converged → stop, go to summary.
If not converged → apply fixes (or get user approval per the skill's rules),
then start the next round.

If the round cap is hit without convergence → stop, go to summary, and flag
that convergence was not reached.

### Interaction model between rounds

- **For skills that auto-fix** (fresh-eyes on code, beads-review): apply fixes
  during the round, then re-run. No user interaction between rounds.
- **For skills that propose changes** (plan-review, fresh-eyes on plans):
  present all proposals at the end of each round. Wait for user approval
  before starting the next round. The user may approve all, approve some,
  or say "stop here."
- **User can interrupt** at any point by saying "stop" or "enough" — go
  directly to the summary.

## Summary

After all rounds complete (or convergence / user stop), present:

```
## Repeat Summary: /<skill>

| Round | Issues Found | High | Med | Low | Fixes Applied |
|-------|-------------|------|-----|-----|---------------|
| 1     | 8           | 3    | 4   | 1   | 8             |
| 2     | 3           | 0    | 2   | 1   | 3             |
| 3     | 0           | 0    | 0   | 0   | —             |

Result: **Converged** after 3 rounds (12 total issues fixed).
```

Or if cap was hit:

```
Result: **Did not converge** after 5 rounds (still finding Med issues).
Consider: different approach, manual review, or splitting the artifact.
```

## Rules

- Never skip the convergence check. Even if the user asked for exactly N rounds,
  stop early if round N-1 was clean.
- The inner skill runs normally — all its rules, output protocols, and checklists
  apply. /repeat is a wrapper, not a replacement.
- Do NOT collapse rounds into one giant review. Each round is a fresh pass —
  the point is that fixing round 1 issues may reveal round 2 issues.
- Between rounds, re-read changed files. Do not rely on memory of the previous
  round's state — the fixes may have changed things.
- Max absolute cap: 7 rounds. Even if the user asks for more, cap at 7 and
  recommend a different approach if still not converging.
- Use extended thinking for convergence analysis.
