---
name: hs-sw-create-arch-diagram
description: Generate or update an architecture digest/diagram (mermaid + verified claims + freshness frontmatter) — for a repo, a feature folder, or a free-text description of the specific view you want
argument-hint: [path | "free-text: the view/flow you want"] [--system|--feature] [--diagram-only] [--append|--replace|--override]
---

# /create-arch-diagram — Architecture Digest Generator

```
┌─ WHAT THIS IS ──────────────────────────────────────────────────────────┐
│ A digest is a CACHE of expensive-to-recompute architecture knowledge.    │
│ Run me in a folder; I write a digest Claude reads INSTEAD of re-deriving  │
│ the architecture from raw code every session. See the wiki-context PLAN   │
│ in joystream/docs/features/wiki-context/ for the full rationale.          │
└──────────────────────────────────────────────────────────────────────────┘
```

Produce an **architecture digest** for whatever you point me at. A digest is
deliberately NOT a full doc — it's the *stable* layer only (boundaries, data flow,
invariants), represented as a mermaid diagram + a verified claims-list + freshness
metadata, so it stays cheap to read and degrades gracefully when code drifts.

---

## When to use

- You want Claude to orient in a codebase fast without re-reading everything.
- A feature or system changed enough that its mental model is worth re-capturing.
- You're onboarding to handed-off code and want a verified map.

**Do NOT use me to document volatile detail** (function bodies, signatures, per-field
behavior). That goes stale instantly and makes the digest a liability. See the
Stable-Layer Test below.

---

## Step 0 — Interpret the request & detect context

`$ARGUMENTS` may contain any mix of: a **path**, **flags**, and **free-flowing text**
describing the specific view the user wants. Parse it into three parts:

1. **Free-text intent** (anything that isn't a path or a flag) — a natural-language
   description of the diagram/view to produce, e.g. *"the auth flow for the new SSO path"*
   or *"how the trigger intake service talks to the queue"*. This is the common
   feature-branch case: the user is mid-work and wants one targeted view, not a full
   regeneration.
   - **No intent given** → produce/refresh the *whole* digest for the target (system or feature).
   - **Intent given** → scope to exactly what they described: identify the relevant code
     area from the description, build that one diagram + its claims, and merge it into the
     right doc (see Step 5). Do not regenerate unrelated sections.

2. **Target folder/file** (default: current working directory) → determines which doc to
   write to and whether it's a system or feature digest:

   | Situation | Digest kind | Output file |
   |-----------|-------------|-------------|
   | **repo root** (has `.git`, or top-level dirs like `backend/`, `frontend/`, `cli/`) | **system** | `docs/architecture/README.md` (create `docs/architecture/` if missing). **Default — decide this automatically, don't ask.** Only fall back to a root `ARCHITECTURE.md` if the repo already has one there. |
   | **`docs/features/<name>/`** (or similar feature dir) | **feature** | `architecture.md` in that folder |

3. **Flags** — `--system`/`--feature` override the kind; `--diagram-only` skips
   claims+frontmatter; `--append`/`--replace`/`--override` force the write mode (otherwise
   it's auto-determined in Step 5).

If kind is genuinely ambiguous and no flag was given, ask one question, then proceed.

---

## Step 1 — Scope the stable layer (the discipline that prevents staleness)

Before writing anything, decide what belongs in the digest using the **Stable-Layer
Test**, applied to every candidate claim:

> **"Would a normal refactor break this claim?"** If renaming a function, reordering
> internal logic, or changing a signature would force you to edit the digest — the claim
> is too detailed. **Leave it out.**

Include only:
- Module/component **boundaries** and responsibilities
- **Who calls whom** / data flow across components
- **Cross-component contracts** ("the CLI reaches the DB only via the backend API")
- **Design invariants** the system is built around

Exclude: function bodies, signatures, line numbers, per-field behavior, anything cosmetic.

**Anchor claims to symbols/modules, never line numbers** — so they survive refactors.

For a **system** digest: scope = the top-level components and how they communicate.
For a **feature** digest: first read ALL docs in the feature folder (PLAN.md,
running-design.md, planning-context/), then scope = the feature's flow + the real code
symbols it touches.

---

## Step 2 — Build the diagram (mermaid by default)

Draw a **mermaid** diagram of the components and data flow. Mermaid is the default —
it's always authorable (just text in a fence), renders in GitHub/VS Code/Obsidian, and
Claude reads it fine as text.

- System digest → a `flowchart` of components + communication paths.
- Feature digest → a `flowchart` or `sequenceDiagram` of the feature's end-to-end flow.
- **Targeted intent** (free-text) → a diagram of *exactly* the described view (e.g. one
  flow or one boundary) and nothing more; its claims in Step 3 cover only that view.

Use ASCII *only* if the user passes `--diagram-only` AND states the digest's audience is a
non-rendering plain-text surface (e.g. terminal output). Otherwise: mermaid.

In mermaid node labels, use `<br/>` for line breaks (renders reliably on GitHub/VS Code);
avoid `\n`, which some renderers show literally.

---

## Step 3 — Build the claims-list and VERIFY every claim

Write a `## Claims` section: a bullet list of the stable-layer assertions. Then, for
**each claim, verify it against the live code via Serena before keeping it**:

- `find_symbol` / `find_referencing_symbols` to confirm a component, call, or boundary
  actually exists as stated.
- If a claim can't be verified against current code → **drop it or fix it.** Never ship an
  unverified claim. (Activate the project with Serena first if needed.)

Each claim should name the symbol/module it concerns so a reader can jump to it.

---

## Step 4 — Stamp frontmatter + trust header

Prepend YAML frontmatter (the freshness anchor — adapted from the research-llm-wiki schema):

```yaml
---
type: architecture          # system | feature
scope: <repo-or-feature-name>
sources:                    # code paths this digest describes (globs)
  - backend/
  - cli/src/
generated_from_sha: <short HEAD sha — run `git rev-parse --short HEAD`>
last_verified: <today's date YYYY-MM-DD>
confidence: high            # high | medium | low — how well-supported the claims are.
                            # NOT freshness. Freshness is DERIVED from generated_from_sha.
---
```

Then the **trust-framing header** as the first body line:

> _Orientation only. Verify against source before modifying anything described here.
> Freshness: generated from `<sha>` on `<date>`._

> **Confidence ≠ freshness.** `confidence` = how well-supported the claims were when
> written. Freshness = whether code moved since, and is *derived* (`generated_from_sha`
> vs current HEAD), never stored as a status — a stored "stale" flag would itself go
> stale. Do not invent a freshness/`stale` field.

---

## Step 5 — Write smartly: create / append / replace / override

Never blindly overwrite. **Determine the write mode**, state it, then apply. If the user
passed `--append`/`--replace`/`--override`, honor that. Otherwise auto-decide:

```
Does the target doc exist?
├─ NO  ──────────────────────────────────► CREATE the doc fresh.
└─ YES ─ read its frontmatter + section headings, then:
   │
   ├─ Whole-digest regeneration (no free-text intent)?
   │   ├─ Existing doc is STALE (its generated_from_sha ≠ current HEAD AND Step-3
   │   │   verification shows its claims no longer match code) ─► OVERRIDE: regenerate
   │   │   the whole doc, refresh sha + last_verified, note what was superseded.
   │   └─ Existing doc still broadly matches code ─► REPLACE the body in place
   │       (refresh content + last_verified), preserving any still-valid sections.
   │
   └─ Targeted view (free-text intent describing one flow/concern)?
       ├─ A section already covers THIS SAME view (match by heading/topic) ─► REPLACE
       │   just that section with the freshly verified version. If that section was
       │   stale, that's an override of the section — say so and bump the stamp.
       └─ No section covers this view ─► APPEND it as a new `## <view>` section, and
           UNION its source globs into the doc's frontmatter `sources`.
```

**Definitions (the three the user thinks in):**
- **append** — add a *new* view/section the doc doesn't have yet. Non-destructive.
- **replace** — swap an *existing* section (or the whole body) with a fresh, verified
  version covering the same scope.
- **override** — replace because the existing content was **stale/wrong**, not merely
  older. Always refresh `generated_from_sha` + `last_verified` and note the supersession.

**Always:**
- State the chosen mode and why before writing (e.g. *"APPEND — no existing section covers
  the SSO auth flow"*, or *"OVERRIDE — `auth-flow` section's sha is 4 commits behind and 2
  claims no longer match `identity.py`"*).
- **Confirm before override/replace of a doc that has NO frontmatter** (i.e. a hand-written
  doc the skill didn't generate) — that's potentially destroying human work. Show what
  you'd change first.
- Keep one file per target — append adds sections, it does not spawn sibling files.
- Refresh `last_verified` on any write; refresh `generated_from_sha` whenever content was
  (re)generated against the current HEAD.

The digest must still **earn its keep**: cheaper to read than the code it summarizes. If
appending keeps growing it, re-audit older sections against the Stable-Layer Test rather
than letting it balloon.

---

## Step 6 — Offer the AGENTS.md router line

After writing, offer to wire consumption (don't inline the digest — point at it):

- **System digest** → add to the repo's root `AGENTS.md`:
  > _System map (read for orientation before exploring code): `docs/architecture/README.md`._
- **Feature digest** → ensure the root `AGENTS.md` notes that feature digests exist at
  `docs/features/<name>/architecture.md` and should be read on demand.

`AGENTS.md` is a **router, not a container** — pointers only, so digests load lazily.

Ask before editing `AGENTS.md`; apply if the user agrees.

---

## Output / report

After writing, report:
1. The file written (path) and the **write mode chosen (create / append / replace /
   override) and why**.
2. How many claims were kept vs dropped-as-unverifiable (the verification did its job).
3. The `generated_from_sha` / `last_verified` stamped.
4. Whether the `AGENTS.md` router line was added (or offered).

---

## Rules

- **Stable layer only.** Apply the Stable-Layer Test to every claim. When unsure, leave
  it out — a smaller true digest beats a larger fragile one.
- **No unverified claims.** Every claim is checked against live code via Serena before it
  ships. This is the core quality gate.
- **mermaid by default**, ASCII only for explicitly non-rendering surfaces.
- **Idempotent.** Re-running upgrades the existing digest; never duplicates files.
- **Trust-framed + stamped, always.** Every digest carries the trust header and freshness
  frontmatter — that's what makes it safe to load every session.
- **Router, not container.** AGENTS.md gets a pointer, never the digest's content.
- Use extended thinking when scoping the stable layer and verifying claims.
