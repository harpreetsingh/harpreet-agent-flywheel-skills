---
name: hs-sw-ux-polish
description: Deep UI/UX scrutiny targeting Stripe-level quality for both desktop and mobile
argument-hint: [component-or-directory]
---

# /ux-polish — World-Class UX Review

```
┌─ THE FLYWHEEL ──────────────────────────────────────────────────────────┐
│ SHAPE → PLAN → REVIEW×N → DECOMPOSE → SPRINT PLAN → EXECUTE → CLOSE   │
│ ★ YOU ARE HERE: Three touchpoints —                                     │
│   1. Planning (UX section in PLAN.md — via /plan-review)                │
│   2. Wave gate (UX lens on frontend waves)                              │
│   3. Sprint close (final UX sweep)                                      │
│ See FLYWHEEL.md for the full development lifecycle.                     │
└─────────────────────────────────────────────────────────────────────────┘
```

Scrutinize every aspect of the application workflow and implementation. Look for
things that are sub-optimal, wrong, unintuitive, or unpolished. Target: the kind
of quality that makes people gasp at how stunning and perfect it is — Stripe-level
premium, where every pixel, every transition, every word feels intentional.

## The bar: "Would someone screenshot this to show a friend?"

Not a checklist exercise — you're hunting for the gap between "works correctly"
and "feels magical." A Stripe-level product doesn't just function; it makes users
feel competent and in control. Every interaction should feel like the product
anticipated what you wanted. If you can't honestly say "this is beautiful and
delightful," keep looking for improvements.

## What "Stripe-level" means concretely

Not a vague aspiration — these are specific benchmarks:

- **Instant feedback on every interaction.** Button press → immediate visual
  response (opacity change, scale) before the async operation completes.
  Optimistic updates everywhere. Never leave the user wondering "did that work?"
- **Graceful degradation in every state.** No blank screens, no raw error dumps,
  no "something went wrong." Every failure has an actionable message and a
  recovery path.
- **Information density without clutter.** Stripe dashboards show a lot of data
  but it never feels overwhelming — spacing, typography, and color do the work.
- **Motion with purpose.** Transitions exist to orient the user (where did this
  come from? where did that go?) — never decorative.
- **Copywriting is product design.** Every label, error message, and empty state
  is written with the same care as the code. No "Error 500" — instead "We couldn't
  save your changes. Try again, or contact support if this persists."
- **Consistency is invisible.** Same patterns everywhere so the user never has to
  re-learn. Same button sizes, same spacing scale, same animation curves.

## Scope

- If `$ARGUMENTS` provided, focus on those components
- Otherwise, review the full application workflow

## Workflow Discovery

Before reviewing individual components, map the user's journey:

1. **Find all routes/pages:** scan router config, page files, navigation components
2. **Identify the core loop:** what does the user do most? (e.g., create → configure → monitor)
3. **Map critical workflows:**
   - Onboarding / first-time experience
   - Core loop (the thing users do daily)
   - Settings / configuration
   - Error recovery (what happens when things go wrong?)
   - Edge cases (empty account, expired session, permission denied)
4. **Trace each workflow** step by step through the code — every click, every
   screen transition, every form submission, every response

## Evaluation Axes

### 1. Usability
Is every interaction intuitive? Can users accomplish goals without thinking?
- Count the clicks/steps for each core task. Can any be eliminated?
- Are destructive actions guarded? (confirm dialog, undo option, or both)
- Is progressive disclosure used? (don't show advanced options upfront)
- Are defaults smart? (pre-fill what you can infer)

### 2. Consistency
Same patterns throughout?
- Mixed patterns (different button styles, inconsistent spacing, varying empty
  states) are jarring
- Audit: do all forms validate the same way? Do all lists paginate the same way?
  Do all modals dismiss the same way?

### 3. Visual hierarchy
Is the most important content prominent?
- Spacing, typography, and color should guide the eye
- Primary action should be immediately obvious on every screen
- Secondary actions should be visually subordinate

### 4. Component state completeness

**Every interactive component must handle ALL 5 states.** This is not optional —
missing states are the #1 source of "unpolished" feel.

| State    | What the user sees                          | Common failure                   |
|----------|---------------------------------------------|----------------------------------|
| Loading  | Skeleton screen (NOT spinner)               | Blank screen or raw spinner      |
| Empty    | Helpful message + CTA to create first item  | Blank area or "No data"          |
| Data     | The actual content                          | — (usually fine)                 |
| Error    | Actionable message + retry button           | Raw error dump or "Error"        |
| Partial  | Some data + inline error for failed portion | Full page error hiding good data |

**Systematic check:** For every component that fetches data, grep for how it
handles each state. If any state is missing, flag it.

### 5. Micro-interactions & feedback
- **Buttons:** press → immediate visual feedback (scale/opacity) → loading state
  → success/error state. Never just "click and wait."
- **Forms:** validate on blur (not just on submit), show inline errors next to
  the field, success state after save
- **Toasts/notifications:** success = auto-dismiss after 3-5s, error = persist
  until dismissed, with action link
- **Confirmation dialogs:** for destructive actions. Include what will happen
  ("Delete 3 agents permanently") not just "Are you sure?"
- **Undo patterns:** prefer undo over confirm where possible (Stripe pattern:
  "Agent deleted" toast with Undo button)

### 6. Copywriting quality
Bad copy kills premium feel. Review every user-facing string:
- **Button labels:** action verbs ("Create agent", not "Submit"), specific not
  generic ("Save changes", not "OK")
- **Error messages:** what happened + what to do about it, not error codes
- **Empty states:** explain value + CTA ("Create your first agent to start
  automating workflows" not "No agents found")
- **Placeholder text:** helpful examples, not field names ("jane@company.com"
  not "Enter email")
- **Confirmation copy:** specific consequences ("This will permanently delete
  Agent X and its 12 runs" not "Are you sure?")
- **Onboarding copy:** benefit-first, not feature-first

### 7. Performance UX
- Optimistic updates — show the result before the server confirms
- Lazy loading — don't load what's off-screen
- Progressive disclosure — show summary first, expand on click
- Perceived speed over raw speed — skeleton screens, staggered animations

### 8. Desktop UX
- Keyboard shortcuts for power users
- Hover states on every interactive element
- Information density — use the space
- Multi-panel layouts where natural
- Drag-and-drop where it reduces clicks

### 9. Mobile UX
- Touch targets min 44px
- Swipe gestures for common actions
- Thumb zones — primary actions in bottom half
- No hover-dependent features
- Responsive breakpoints that actually redesign, not just shrink

### 10. Accessibility
- Contrast ratios WCAG AA minimum (4.5:1 text, 3:1 UI components)
- Focus indicators visible and consistent
- Screen reader labels on all interactive elements
- Reduced-motion support (`prefers-reduced-motion`)
- Keyboard navigation for all workflows

### 11. Dark mode & theming
- If dark mode exists: is it consistent? (no bright flashes, proper contrast)
- If no dark mode: flag as a gap for premium feel
- Are colors from a design token system or hardcoded hex values?

### 12. CLI UX
CLI is a first-class interface, not an afterthought:
- Does every feature accessible via UI/API also have CLI commands?
- Do all commands support `--json` for agent/machine consumption?
- Is human-friendly output the default? (rich formatting, colors, tables)
- Are error messages actionable ("missing --workspace flag") not cryptic?
- Is help text (`--help`) complete and useful?
- Are command names intuitive and consistent? (`<noun> list|show|create|update|delete`)
- Is output scannable? (headers, whitespace, alignment for human mode;
  parseable structure for `--json` mode)

## Process

1. **Discover workflows** (see Workflow Discovery above)
2. Trace the component hierarchy: pages → layouts → components → primitives.
   Since you can't see the running app, reconstruct the visual structure from code.
3. **Desktop pass** — walk through every critical workflow as a desktop user.
   Evaluate all 12 axes. Think: large viewport, mouse + keyboard, hover states,
   information density, multi-panel layouts, keyboard shortcuts. Desktop users
   expect power and density. Tag all desktop findings with `[D]`.
4. **Mobile pass** — walk through every critical workflow AGAIN as a mobile user.
   This is a completely separate review, not "does desktop shrink OK." Think:
   thumb zones, touch targets (44px min), swipe gestures, bottom-half primary
   actions, no hover-dependent features, responsive breakpoints that REDESIGN
   (not just reflow). Mobile users expect speed and reachability. Tag all mobile
   findings with `[M]`.
5. For each component: check the 5-state matrix (loading, empty, data, error, partial)
6. Evaluate each axis above
7. **Output the Issues Table** (see Output Protocol below) — include `[D]`/`[M]`/`[DM]`
   tags so desktop and mobile issues are visually distinct
8. **Walk through issues one-at-a-time** with the user
9. Implement changes directly OR create beads if the scope is large
10. **Final status table** after all issues are walked

## Output Protocol

All review output follows a three-phase structure. Do NOT dump a wall of findings.

### Phase 1 — Issues Table

After completing your review, present ONLY a compact table:

```
| #  | Tag  | Handle              | Description                                       | Crit | Status |
|----|------|---------------------|---------------------------------------------------|------|--------|
| 1  | [DM] | no-loading-skeleton | Agent list shows spinner instead of skeleton screen | High | ✗      |
| 2  | [DM] | delete-no-undo      | Deleting an agent has no undo — just a confirm box  | Med  | ✗      |
| 3  | [D]  | no-keyboard-nav     | Dashboard has no keyboard shortcuts for power users | Med  | ✗      |
| 4  | [M]  | touch-target-small  | Action buttons are 32px — below 44px minimum        | High | ✗      |
```

Column definitions:
- **#**: Sequential number
- **Tag**: `[D]` desktop-only, `[M]` mobile-only, `[DM]` both modalities
- **Handle**: 2-5 word slug
- **Description**: 1-2 sentences
- **Crit**: `High` / `Med` / `Low`
- **Status**: `✗` (open) or `✓` (addressed)

Sort by criticality (High first), then by workflow order.

After presenting the table, say:
> "Ready to walk through each issue. Say **go** to start from #1, or pick a number."

### Phase 2 — One-at-a-Time Walkthrough

For each issue:
1. State the handle and issue number
2. Show the full analysis: what's wrong, where in code, which Stripe benchmark it violates
3. Propose the specific fix with code changes or bead structure
4. Wait for approval before implementing
5. Mark status `✓` or deferred
6. Move to the next issue

Do NOT present multiple issues at once. One issue per response.

### Phase 3 — Final Status Table

After all issues walked, present the updated table with final statuses.
If any remain `✗`, call them out and ask if the user wants another pass.

## Rules

- **Desktop and mobile are separate modalities** — each gets its own full review
  pass. Desktop optimizes for density, keyboard flow, and hover richness. Mobile
  optimizes for thumb reachability, touch precision, and speed. A feature that
  works on desktop and "also works" on mobile is not mobile-optimized.
- Be specific. Not "improve spacing" but "increase card padding from 12px to 16px
  for better visual breathing room."
- Every finding must reference a concrete Stripe-level benchmark, not just
  "this could be better."
- The 5-state matrix is non-negotiable. Every data-fetching component must
  handle all 5 states.
- When referencing best practices, explain the principle — don't just name-drop.
- **Push past "correct" to "delightful."** A button that works is table stakes.
  A button with the right press feedback, loading state, success animation, and
  hover treatment is Stripe-level. If the interaction doesn't feel crafted,
  flag it — even if it's technically functional.
- Use extended thinking for deep UX analysis.
