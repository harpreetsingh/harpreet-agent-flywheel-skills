---
name: hs-sw-shape
description: Run a Shape Up shaping interview (5 rounds) and produce a pitch.md for a feature
argument-hint: [feature-name-or-gh-issue]
---

# /hs-sw-shape — Shape Up Shaping Interview

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ ★SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE  │
│ ★ YOU ARE HERE: Entry point. Shape the problem before building.         │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Run a 5-round shaping interview for a feature and write the output to
`docs/projects/features/<slug>/pitch.md` in the current project.

**Usage:**
```
/hs-sw-shape #42
/hs-sw-shape "skill search redesign"
/hs-sw-shape   ← prompts for feature name
```

---

## What Shaping Is

Shaping is NOT spec writing. It's finding the right level of abstraction:
- **Too vague:** "improve onboarding" — no one knows what to build
- **Too detailed:** "add a blue 40px button at coordinates..." — leaves no room for builders
- **Shaped:** Clear problem, fixed appetite, rough solution direction, rabbit holes named

The output is a pitch — enough for a senior engineer to start building without asking
clarifying questions, but not so detailed it constrains every decision.

---

## Process

### Step 0 — Setup

If `$ARGUMENTS` is empty, ask: "What feature are we shaping?"

Derive a slug from the feature name/ID (lowercase, hyphens, no spaces).
Example: "#42: Skill Search Redesign" → `skill-search-redesign`

Check if `docs/projects/features/<slug>/pitch.md` already exists. If so, ask: "A pitch
already exists at that path. Continue from it, or start fresh?"

### Step 1 — Open the interview

Tell the user:
> "Let's shape **[feature name]**. I'll ask questions across 5 rounds to build up
> the pitch. Answer as much or as little as you know — we'll fill gaps together.
>
> **Round 1 of 5: The Problem**"

Then ask ALL of the following in one message (don't drip one at a time):

- What's the user pain this solves? Describe it as a concrete situation, not an
  abstraction. ("A user tries to X and can't because Y" is better than "UX is poor")
- Who specifically has this problem? (All users? Power users? CTOs? New signups?)
- How often do they hit it?
- What do they do today instead? (workaround, give up, use a different tool)
- Why does this matter NOW? What changed, or what deadline/context makes this urgent?

### Step 2 — Appetite

Respond to their Round 1 answers with a brief reflection (1-2 sentences confirming
you understood the problem), then:

> "**Round 2 of 5: Appetite**"

Ask:
- How much time are you willing to spend on this? (This is a CONSTRAINT, not an
  estimate. The solution must fit in this budget.)
- If the ideal solution takes 3x longer, would you descope or skip it?
- Are there any hard deadlines (demo, customer commitment, release)?

Flag if the stated appetite seems mismatched with the scope they described in Round 1.
Example: "You described a fairly complex workflow — 2 days feels tight. Are you
imagining a minimal version, or is the scope narrower than I'm reading?"

### Step 3 — Solution

Reflect on appetite (1 sentence), then:

> "**Round 3 of 5: Solution**"

Ask:
- What's your instinct on the approach? (Even a rough direction: "add a search bar",
  "replace the modal with a sidebar", "new API endpoint + UI component")
- What does the happy path look like in 3-5 user steps?
- What's the ONE thing that makes this solution valuable — the core interaction?
- What does the user see/do differently after this ships vs before?
- **CLI surface:** What CLI commands should expose this feature? (Every feature
  needs a CLI interface, not just a UI. Think: what would an agent or power user
  run from the terminal? All commands must support `--json` for machine consumption.)

After they answer: synthesize a rough solution sketch in your response. Use an ASCII
diagram if it helps clarify the flow — this sketch is spoken in conversation, and the
terminal can't render mermaid. (The diagram in the written pitch.md is mermaid; see
the template below.) Ask: "Does this capture the direction, or am I off?"

### Step 4 — Rabbit Holes

Reflect on the solution (1-2 sentences), then:

> "**Round 4 of 5: Rabbit Holes**"

Based on everything you've heard, proactively identify 3-5 things that look like
they're in scope but could blow up the appetite. Present them and ask the user to
react: in scope, out of scope, or defer to follow-up?

Good rabbit holes to probe:
- Edge cases that require different UI/logic branches
- Error states and failure modes
- Permissions / multi-user / multi-tenant implications
- Data migration or backward compatibility
- Performance at scale
- Mobile / responsive concerns
- Anything that sounds like "while we're at it..."

For each one the user flags as out-of-scope, note it as an explicit No-Go.

### Step 5 — No-Gos and Confirm

> "**Round 5 of 5: No-Gos and Final Check**"

Present a consolidated list of everything that came up as explicitly out of scope.
Ask the user to confirm or add to it.

Then ask: "Anything else you want to call out before I write the pitch?"

### Step 6 — Write the Pitch

Tell the user: "Got it. Writing the pitch now."

Create the directory if it doesn't exist: `docs/projects/features/<slug>/`

Write `docs/projects/features/<slug>/pitch.md` using the structure below. Do NOT ask for
more input at this stage — synthesize from the conversation.

---

## Pitch Output Format

```markdown
# Pitch: [Feature Name]

**GitHub Issue:** [#number from arguments, or "—" if not provided]
**Shaped by:** [ask user if not obvious from context]
**Date:** [today's date]
**Appetite:** [X days / X weeks — as a constraint]

---

## Problem

[Concrete description of the user pain. Who, what situation, how often,
what they do today. 2-4 sentences. No abstract language.]

## Appetite

[The time budget as a hard constraint. What descoping would happen if
the work runs long. Any hard deadlines.]

## Solution

[The rough approach. What the user does differently. Happy path in 3-5 steps.
The one core interaction that makes this valuable.

Include a mermaid diagram if the flow has more than 2 steps or involves
multiple components. Example:]

\`\`\`mermaid
flowchart TD
  A[User types in search bar] --> B[Debounced query<br/>/api/skills/search]
  B --> C[Results appear inline<br/>no page reload]
  C --> D[User clicks skill → added to agent]
\`\`\`

[Fat-marker level of detail. Leave implementation decisions to the builder.]

## Rabbit Holes

[List each risk with an explicit decision:]

- **[Risk]:** [Why it could blow up scope] — [Decision: in scope / out of scope / defer]
- ...

## No-Gos

[Explicit list of things NOT being built in this cycle:]

- Not building X (follow-up)
- Not supporting Y (v0.2)
- ...
```

---

## After Writing

Create `docs/projects/features/<slug>/planning-context/` if it doesn't exist.

If the user shared any research, screenshots, competitor analysis, PRD drafts,
or reference material during the shaping interview, save them into
`planning-context/`. If the user referenced external links or docs, note them
in a `planning-context/sources.md` file.

Tell the user the pitch was written at `docs/projects/features/<slug>/pitch.md`, then say:

> "Next steps:
> 1. Drop any research, screenshots, or competitor analysis into
>    `docs/projects/features/<slug>/planning-context/` — this is the evidence bag for
>    planning decisions
> 2. Link this pitch in the GitHub issue description
> 3. Add the `ready-to-bet` label on the GitHub issue
> 4. When the team co-opts it, run `/hs-sw-plan-draft docs/projects/features/<slug>/` to produce PLAN.md"

---

## Quality Bar

A good pitch passes this checklist. Verify before writing:

- [ ] Problem is a concrete user situation, not an abstraction
- [ ] Appetite is stated as a constraint (not just an estimate)
- [ ] Solution has a happy path and at least one mermaid diagram (if flow is non-trivial)
- [ ] Every rabbit hole has an explicit decision (in/out/defer)
- [ ] No-Gos are explicit, not implied
- [ ] CLI commands are specified (what the user/agent runs from terminal, with `--json`)
- [ ] A senior engineer could start building without asking clarifying questions

If any item fails, go back and ask a focused follow-up question before writing.

---

## Tone and Conduct

- Ask multiple questions per round in one message — don't drip single questions
- Reflect back what you heard before asking the next round — shows you're synthesizing
- Push back on vague answers: "Can you give me a concrete example?"
- Flag mismatches between scope and appetite directly — don't paper over them
- Don't lead the witness: ask open questions, not "so you want a dropdown, right?"
- Keep the conversation moving — 5 rounds should feel focused, not exhausting
