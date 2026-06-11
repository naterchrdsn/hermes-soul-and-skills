---
name: session-recall-and-continuity
description: Reconstruct prior work, rolling task lists, and unresolved items from session history without asking the user to restate them.
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [session-recall, continuity, todo-recall, context-compaction, history]
    related_skills: [writing-plans, verification-before-completion, affine]
---

# Session Recall and Continuity

Use this when the user asks things like:
- "what was on my to-do list yesterday?"
- "where did we leave this?"
- "recall my rolling to-do list"
- "what's left from the stuff we worked on?"
- any case where prior context may have been compacted, interrupted, or split across sessions.

## Goal

Recover the real prior state from session history, then report:
1. the original list or closest authoritative version,
2. what was completed,
3. what is partial/in progress,
4. what is still open.

Do not make the user restate their own backlog if the transcript can tell you.

## Core workflow

### 1) Search for the user's own phrasing first

Start with the exact phrase the user used if it is distinctive.

### 2) Prefer the session where the list was created or preserved

Look for:
- the message where the user enumerated tasks,
- a `todo(...)` tool call that normalized/deduped the list,
- a compaction summary that explicitly says the active task list was preserved.

### 3) Reconstruct status from evidence, not vibe

For each task, classify only from transcript evidence:
- **completed**
- **partial / in progress**
- **pending**
- **blocked**

Do not upgrade a task to done just because planning happened.

### 4) Merge duplicates and aliases

Users often repeat the same task with slightly different wording.

### 5) Report the answer in two layers

Preferred output:
- short direct answer first
- then grouped status buckets:
  - Done
  - Partial / in progress
  - Still open

## Pitfalls

- Do not stop at the first approximate match if the user says you found the wrong session.
- Do not rely on broad topic search alone when the user is asking about a named list.
- Do not confuse planning with completion.
- Do not ignore compaction summaries.
- Do not flatten partials into pending.

## Verification checklist

Before answering, make sure you can point to:
- where the original list came from,
- at least one message or tool-backed fact for each claimed completion,
- any blocker that explains why an item is still open.
