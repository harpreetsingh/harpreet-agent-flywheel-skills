---
name: hs-sw-beads-label
description: Add labels to an epic and all the tickets created under it (or to an explicit list of bead IDs)
argument-hint: <epic-id | bead-ids...> --labels label1,label2,...
---

# /beads-label — Bulk-label an epic and its tickets

Adds one or more labels to an epic and every ticket created under it. Use this
when beads were created without `--labels` (or you want to add a tag later — a
feature name, a GH ref, a sprint id). Pairs with `/hs-sw-beads-create`, which can
apply labels at creation time and sets `--parent <epic>` so the set is enumerable.

## Arguments

- **Target** (one of):
  - `<epic-id>` — resolves the epic **plus all its children** via
    `bd list --parent <epic-id>`. This is the normal path.
  - `<bead-id> <bead-id> ...` — an explicit space- or comma-separated list of bead
    IDs (use when you want a subset, or when the beads have no parent epic).
- `--labels label1,label2,...` (**required**) — comma-separated labels to add.

If `--labels` is missing, or no target is given, print usage and stop.

## Process

### Step 1 — Resolve the target bead set

- If the target looks like a single epic ID, build the set as:
  ```bash
  bd list --parent <epic-id> --json    # all children
  ```
  and include the epic ID itself. Collect all IDs.
- If the target is an explicit list of IDs, use those verbatim.
- If an epic resolves to **zero** children (beads weren't created with `--parent`):
  report that, and ask the user to pass the ticket IDs explicitly, or re-run
  showing `bd dep tree <epic-id>` so they can identify the set. Do NOT guess.

Show the resolved set before changing anything:
```
Will add labels [org-management, gh-269] to 14 beads:
  joystream-ep01  Epic: Org management
  joystream-a1b2  Add org schema
  ... (12 more)
```

### Step 2 — Apply the labels

For each bead in the set:
```bash
bd update <id> --add-label label1,label2
```
`--add-label` is additive — it never removes existing labels. Run for every bead;
if any single update fails, note it and continue (don't abort the batch).

### Step 3 — Verify and report

```bash
bd list --label label1 --json    # confirm count matches the target set
```

Report:
```
✓ Labels added.
  • 14/14 beads now carry: org-management, gh-269
  • Verified: bd list --label org-management → 14 beads

(If any failed:)
⚠ 1 bead failed — retry:  bd update joystream-xxxx --add-label org-management,gh-269
```

## Rules

- `--add-label` only — never `--set-labels` (which would wipe existing labels like
  `qa-passed`, `caught:*`, `tier:*`). Additive only.
- Always show the resolved set and wait for nothing in dry obvious cases, but if
  the set is large (>25 beads) or the labels look risky (contain `:` namespacing
  that collides with system labels like `caught:` / `qa-passed`), confirm with the
  user before applying.
- Idempotent: re-running with the same labels is a no-op per bead (bd dedups).
