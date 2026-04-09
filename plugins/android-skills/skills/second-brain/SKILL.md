---
name: second-brain
description: Use when you or the user discover something worth remembering — a decision rationale, a hard-won learning, a useful pattern, an anti-pattern, a mental model, or a reusable runbook. Also use when the user explicitly asks to save something to the second brain, or when searching it for prior knowledge.
---

# Second Brain

Personal knowledge base at `~/Documents/second-brain/`. Not Android-specific — covers any domain.

## When to Invoke

- You solved a non-obvious problem and the root cause or fix would be valuable next time
- A decision was made with meaningful trade-offs worth recording
- The user says "remember this", "save this", "write this down" (and it's knowledge, not a collaboration preference — those go to auto-memory)
- You or the user need to search for prior decisions, learnings, or patterns
- The user explicitly asks to save or search the second brain

**Do not invoke** for ephemeral task context, collaboration preferences (use auto-memory), or things already captured in code/commits.

## Structure

```
~/Documents/second-brain/
├── INDEX.md              # Quick-scan index — read this first
├── decisions/            # Why X over Y — ADR-style
├── learnings/            # Gotchas, debugging war stories
├── evaluations/          # Library/tool comparisons
├── patterns/             # Reusable solutions, scaffolding
├── runbooks/             # Step-by-step procedures
├── mental-models/        # Decision frameworks
└── anti-patterns/        # "Don't do X because Y"
```

## Writing an Entry

### 1. Pick the right folder

| What you learned | Folder |
|---|---|
| "We chose X over Y because Z" | `decisions/` |
| "X breaks when Y — here's the fix" | `learnings/` |
| "Library A vs B vs C" | `evaluations/` |
| "Here's how I structure X" | `patterns/` |
| "Step-by-step to do X" | `runbooks/` |
| "How I decide between X and Y" | `mental-models/` |
| "Never do X because Y" | `anti-patterns/` |

### 2. Create the file

**Filename:** `{topic}-{subtopic}.md` — descriptive, grep-friendly, no date prefixes.

**Format — conclusion first, context second:**

```markdown
---
tags: [tag1, tag2]
confidence: high          # high | medium | low
created: YYYY-MM-DD
---

One-line summary of the finding.

## Decision / Finding / Rule

What was decided or discovered. Lead with the answer.

## Context

Why this came up. What problem was being solved.

## Consequences / Trade-offs

What this implies going forward. What you gave up.
```

- Keep it short and factual — no fluff
- One concept per file
- Use relative markdown links to cross-reference other entries: `[related](../learnings/other-topic.md)`
- `confidence: low` for things you're still testing; update to `high` once validated

### 3. Update INDEX.md

Add one line under the appropriate section:

```markdown
- [Title](folder/file.md) — one-line summary
```

Keep INDEX.md under 200 lines. It's the first thing read when searching.

## Searching

When looking for prior knowledge:

1. **Read `INDEX.md` first** — fastest scan of all topics
2. **Grep by tag** if INDEX doesn't surface it: `tags: \[.*topic.*\]`
3. **Grep by content** as a last resort

## Asking the User

Before writing, briefly tell the user what you want to save and which folder. Don't ask for permission on every field — just confirm the gist is right. Example:

> "That Compose recomposition gotcha is worth saving to `learnings/`. I'll note that `derivedStateOf` inside `remember` skips recomposition but `derivedStateOf` alone doesn't. Sound right?"
