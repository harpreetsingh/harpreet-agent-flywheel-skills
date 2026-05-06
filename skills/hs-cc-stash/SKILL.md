---
name: hs-cc-stash
description: Save session context to tmp/session-resume.md before compaction or ending a session. User provides guidance on what matters.
argument-hint: [~Nm] [topic <keyword>] [free-text focus]
---

# /stash — Save session context before compaction

The user is about to compact or end a session. They have provided guidance on what matters:

**Raw arguments:** $ARGUMENTS

## Parse arguments

Before doing anything, parse `$ARGUMENTS` for two optional modifiers:

| Modifier | Pattern | Example | Meaning |
|----------|---------|---------|---------|
| **Time window** | `~Nm` or `~Nh` | `~5m`, `~1h` | Focus on conversation from roughly N minutes/hours ago, not just the most recent exchange |
| **Topic focus** | `topic <keywords>` | `topic caching strategy` | Scan the conversation for where this topic was discussed, whenever it occurred |

These can be combined. Everything else is free-text focus (existing behavior).

**Examples:**
```
/stash                                → full session state (use judgment)
/stash ~5m caching strategy           → go back ~5 min, capture that context
/stash topic auth token refresh       → find where we discussed auth tokens
/stash ~10m topic error handling      → ~10 min ago, specifically error handling
/stash the prisma migration approach  → free-text focus (existing behavior)
```

**Parsing rules:**
1. If `~Nm` or `~Nh` is present: extract it, set **time window**. Remove from remaining text.
2. If `topic <words>` is present: extract everything after `topic` (up to end or next modifier) as **topic keywords**. Remove from remaining text.
3. Whatever remains is **free-text focus** (existing behavior).
4. If nothing remains after extraction, that's fine — the modifiers are the focus.

## How to apply modifiers

- **Time window set:** Scan backwards in the conversation to approximately that time window. Focus your capture on what was being discussed THEN, not just the most recent exchange. The conversation's most recent messages are still included for state, but the time-windowed content gets priority and detail.
- **Topic set:** Scan the ENTIRE conversation for where this topic came up. It may have been discussed across multiple points. Gather all relevant context for that topic — decisions, options considered, conclusions reached, open questions.
- **Both set:** Go back to the time window AND filter for the topic. This is the most precise: "~10 minutes ago we were talking about X."
- **Neither set:** Full session state using your judgment (existing behavior).

Your job: write a structured session resume to the project stash directory.

## Stash location

Always write to `tmp/stash/` relative to the **project root** (the directory containing
`.git`). Create `tmp/stash/` if it doesn't exist. Ensure `tmp/` is in `.gitignore`.

**File naming:** `YYYY-MM-DD-HHMMSS-session-resume.md` using current date and time.

```
<project-root>/
  tmp/
    stash/
      2026-05-06-143022-session-resume.md
      2026-05-05-091500-session-resume.md
      2026-05-03-170845-session-resume.md
```

Do NOT overwrite previous stashes. Each `/stash` invocation creates a new dated file.
This gives the user a history of session snapshots.

**Cleanup:** If there are more than 10 stash files, delete the oldest ones beyond 10.
Keep it bounded but useful.

## What to capture

Use the parsed focus (time window, topic, free-text) to prioritize what goes into the resume. Not everything matters equally — the user told you what's relevant. Build the file around their focus, then fill in supporting context.

## Required sections

Write the file in this exact structure:

```markdown
**Where we are:**
[1-3 sentences. Current state of the work. What phase, what's done, what's in-flight.]

**Branch & git state:**
[Current branch name, last commit SHA and message, any uncommitted changes]

**Key decisions made this session:**
[Bulleted list. Only decisions that would be lost without this file. Include the WHY, not just the WHAT.]

**Key files touched or referenced:**
[Bulleted list of file paths with 1-line descriptions of why they matter.]

**Bead status:**
[Any beads that are in_progress or were closed this session. Include IDs and titles.]

**Open threads / unresolved:**
[Bulleted list. Things we discussed but didn't finish. Disagreements. Parked ideas.]

**Immediate next step:**
[One concrete action. Not a list of 10 things. THE thing to do when context resumes.]

**After that:**
[2-5 follow-up items in priority order.]
```

## Rules

- Keep the total file under 100 lines. Brevity is the point — this gets injected into a fresh context window.
- Do NOT pad with generic summaries. Every line should carry information that would be lost without it.
- Do NOT include things that are already in committed docs or CLAUDE.md — only session-specific context.
- If the user gave no arguments, review the full conversation and use your judgment on what matters most.
- When a time window is set, you're estimating — conversations don't have timestamps visible to you. Use message count as a proxy (~1 message per minute is a reasonable heuristic). The point is "go back a bit" not "hit an exact timestamp."
- When a topic is set, be thorough — scan the full conversation, not just recent messages. The user is asking you to find something specific that may have happened a while ago.
- Never overwrite previous stashes — each invocation creates a new dated file.
- Clean up stash files beyond the 10 most recent.
