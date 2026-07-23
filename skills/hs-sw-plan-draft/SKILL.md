---
name: hs-sw-plan-draft
description: Synthesize decision docs, PRDs, and notes into a structured PLAN v1 using the canonical template
argument-hint: [source-dir-or-files]
---

# /plan-draft — Generate a PLAN from source docs

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → ★PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Synthesize pitch + research into a structured PLAN.     │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Read all documents in `$ARGUMENTS` (a directory or space-separated file paths)
and synthesize them into a single, coherent PLAN following the canonical template.

## Process

1. **Read the template.** Find `templates/PLAN-TEMPLATE.md` in the
   claude-workflow-skills repo (check `~/.claude/skills/hs-sw-plan-draft/` symlink
   to locate the repo root, or search for it). This defines the target structure.

2. **Read all source documents.** Read every file in the provided directory
   (or listed files). These may be:
   - Decision docs (architectural choices with rationale)
   - PRDs (product requirements)
   - Meeting notes or brainstorm docs
   - Existing partial plans
   - README or handoff docs
   - **`planning-context/`** — if this subdirectory exists, read everything in
     it. This is the evidence bag: research, competitor analysis, PRD drafts,
     screenshots, reference material that informed the pitch. These inputs
     ground the plan in evidence rather than assumption.

3. **Synthesize, don't concatenate.** The goal is a single coherent plan, not
   a paste-together of source docs. For each template section:
   - Pull relevant content from across all source docs
   - Resolve contradictions (flag if you can't resolve)
   - Fill gaps with reasonable inferences (flag what you inferred)
   - Maintain traceability: note which source doc informed each section

4. **Write the PLAN.** Output to `docs/projects/features/<feature-name>/PLAN.md`.
   If the input is a feature directory, write PLAN.md inside it.
   Ask the user for the output path if unclear.

5. **Flag gaps.** After writing, explicitly list:
   - Template sections you couldn't fill (missing info in source docs)
   - Contradictions between source docs you couldn't resolve
   - Inferences you made that the user should validate
   - Open Questions (move these to the Open Questions section)

## Template Scaling

The template scales by scope. Not every section needs equal depth:

| Scope | Heavy sections | Light sections |
|-------|---------------|----------------|
| **Feature** | Deliverables, Context | Architecture, Deployment |
| **Dot release** | All sections roughly equal | — |
| **Major release** | Architecture, Context, Deliverables | — (all sections full) |
| **Architecture** | Architecture, Key Decisions | Implementation (may be TBD) |

Collapse sections that aren't relevant — don't fill them with filler.

## Diagrams Are Required

**A PLAN without at least one diagram is incomplete.** Do not consider the plan
done until the diagram requirement is satisfied.

Every plan needs a minimum of one diagram. Complex features need more.

**Use mermaid.** The plan is a markdown file read in GitHub/VS Code/Obsidian, where
mermaid renders — and it's still plain text in a fence everywhere else. No external
tools, no image links. Diagrams live inline in the plan.

In mermaid node labels use `<br/>` for line breaks; avoid `\n`, which some renderers
show literally. If you summarize the plan on screen in conversation, redraw that
summary in ASCII — the terminal doesn't render mermaid.

| Feature complexity | Required diagrams |
|--------------------|-------------------|
| Simple (1 component, linear flow) | 1 — architecture/data flow |
| Medium (2-3 components, branching) | 2 — architecture + user flow or sequence |
| Complex (multi-component, async, multi-actor) | 3+ — architecture, user flow, sequence, data model |

**Required diagram types by what the feature does:**

- **Any feature with a user-facing flow** → user flow diagram (what the user sees/does step by step)
- **Any feature with an API or data layer** → architecture diagram (components + data flow)
- **Any feature with async operations or multi-step sequences** → sequence diagram (who calls who, in order)
- **Any feature that modifies data models** → data model diagram (tables, fields, relationships)

**Format:**
```
Component A → Component B
     ↓               ↓
  Output A        Output B
```

If you find yourself writing a paragraph that describes a flow, stop and draw it instead.
Paragraphs describing flows are a red flag — they hide complexity and are hard to review.

## Rules

- The plan must be self-contained. An agent reading only the plan should be able
  to implement it without reading the source docs.
- Preserve key decisions and their rationale — this is the most valuable content
  in decision docs. Don't summarize away the WHY.
- Write in the same voice as the source docs. Don't sanitize personality.
- Use extended thinking for synthesis.
- **Prior art check required.** Before drafting, run:
  1. `cm context "<feature name>" --json` — get rules, anti-patterns, and
     past solutions from the CM playbook
  2. `cass search "<feature>" --json --limit 5` — find past sessions that
     touched this area (prior implementations, debugging, design decisions)
  3. Serena (`search_symbols`, `find_symbol`) + grep — find existing code
  Include a "What Exists Already" subsection in the plan mapping prior
  sessions AND existing code to planned changes — with verified file paths.
  This prevents agents from rebuilding during sprints. A plan that proposes
  new code without checking CM, CASS, and the codebase is incomplete.
- **No diagrams = incomplete plan.** Do not write "diagram TBD" — draw it now.
- **CLI is a first-class deliverable.** Every feature that has an API or UI MUST
  also specify its CLI commands. Include a "CLI Interface" section in the plan
  listing: command names, flags, arguments, and expected output. All commands
  MUST support `--json` for agent/machine consumption. A plan without CLI
  commands for its features is incomplete — same as missing diagrams.
- **Testing strategy is a first-class deliverable.** Every plan MUST include a
  "Testing Strategy" section that identifies:
  - Which deliverables need tests (any with an API, business logic, UI component,
    or data model)
  - What kind of tests (unit, integration, CLI, e2e)
  - Key test scenarios for each deliverable (happy path + critical edge cases)
  - What should NOT be mocked (real DB, real API calls) vs. boundary mocks
  
  This section feeds directly into `/beads-create` which creates TDD test beads.
  A plan without a testing strategy will produce beads without test coverage —
  and untested code ships stubs. Same severity as missing diagrams or CLI.

## Next Steps

After generating the PLAN, tell the user:
- "Run `/plan-review` to refine this plan (typically 4-5 rounds of review)."
- "Then `/beads-create` to decompose into executable tickets."
