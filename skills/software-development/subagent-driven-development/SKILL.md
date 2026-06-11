---
name: subagent-driven-development
description: "Execute plans via delegate_task subagents (2-stage review)."
version: 1.1.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
metadata:
  hermes:
    tags: [delegation, subagent, implementation, workflow, parallel]
    related_skills: [writing-plans, requesting-code-review, test-driven-development]
---

# Subagent-Driven Development

## Overview

Execute implementation plans by dispatching fresh subagents per task with systematic two-stage review.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration.

## When to Use

Use this skill when:
- You have an implementation plan (from writing-plans skill or user requirements)
- Tasks are mostly independent
- Quality and spec compliance are important
- You want automated review between tasks

**vs. manual execution:**
- Fresh context per task (no confusion from accumulated state)
- Automated review process catches issues early
- Consistent quality checks across all tasks
- Subagents can ask questions before starting work

## Variant: Multi-Phase Git-Safe Parallel Execution

When tasks touch the same git repository, parallel execution must respect git state. The pattern:

1. **Sequential phases** for anything that mutates shared config files (`angular.json`, `package.json`, `tsconfig.spec.json`) or git history (commits). Only one agent at a time.
2. **Parallel phases** for file edits to independent files with no shared git index operations. Multiple agents edit different files simultaneously; the parent commits once all return.
3. **Never parallelize agents that commit** — concurrent commits to the same branch produce conflicts. All agents in a parallel batch must NOT commit; the parent commits after all return.

Practical sequencing for a large migration:
```
Phase 1 (parent): create worktree / branch
Phase 2 (1 agent): infra changes (config files, package.json) → commits
Phase 3 (3 agents, parallel): file edits in independent spec files → NO commits
Phase 4 (parent): commit all Phase 3 changes together
Phase 5 (1 agent): additional file work → parent commits
```

The parent agent's `git add -A && git commit` after a parallel phase is the key. Brief the parallel agents explicitly: "DO NOT commit — parent agent will commit."

## Variant: Parallel Extraction Fan-Out (no review loop)

A lighter variant for independent extraction/port/build tasks where the spec is clear, the source material is readable, and you want maximum throughput without review loops.

**When to use instead of the full plan-execute-review flow:**
- Tasks are truly independent (different output files, no shared state)
- Each task has a clear "read these source files, produce this output" shape
- You can write a precise brief for each task up front
- Review is by inspection (run tests, read output) not by spec-checklist

**The pattern:**

1. Read all relevant source files YOURSELF before dispatching — subagents can't read AFFiNE/MCP docs, and you don't want them guessing at source structure.
2. Dispatch all tasks in a single `delegate_task(tasks=[...])` call (up to 3 concurrent).
3. Write context briefs that include:
   - Exact source file paths to read (and their key sections if large)
   - Exact output paths to create
   - Language/framework constraints
   - Critical invariants (e.g. "zero runtime deps", "thread-safe", "test must pass")
   - Python interpreter path if non-standard (`~/DEV/hermes-agent/.venv/bin/python`)
4. After all return, spot-check: run tests, read key output files, verify integration points.

**Example brief structure:**
```
goal: "Port the Foo class from ~/DEV/src/foo.ts to Python at ~/DEV/out/foo.py"
context: """
SOURCE (read fully): ~/DEV/src/foo.ts
  - FooClass with methods: bar(), baz()
  - Key invariant: ...

DELIVERABLES:
1. ~/DEV/out/foo.py — Python port with FooClass
2. ~/DEV/out/tests/test_foo.py — pytest tests covering all public methods

DESIGN NOTE: Use Python's <module> instead of regexes (cleaner, correct for edge case X)
Run tests: ~/DEV/hermes-agent/.venv/bin/python -m pytest tests/ -v
"""
toolsets: ["terminal", "file"]
```

**What makes good briefs for this pattern:**
- Source file paths are explicit (subagents cannot search well; give them the path)
- Output paths are explicit
- Test command is explicit
- Key design decisions are stated as directives, not left for the subagent to infer
- One brief per task — don't combine two extraction tasks into one brief
- **For migration tasks (API rewrites, pattern replacements):** run all relevant grep/inventory commands YOURSELF before dispatching, and paste the actual output into the brief. Do NOT rely on the agent to discover what needs changing. Agents routinely miss uncommon call sites (e.g. `spyOn(console, 'warn')` in a Jasmine→Jest migration, `toBeTrue/toBeFalse` matchers, `.calls.*` matchers). If your grep found 17 occurrences, the brief should list all 17.

**What to verify after all return:**
- Run tests for each output: `pytest <new-package>/tests/ -q`
- Read the key output file (not just the subagent's self-report)
- Check for import-time errors by importing the new module
- For harness plugins specifically: run the discovery + load + initialize sequence (see harness-integration-shape references)
- **For migration tasks:** re-run the same inventory greps you ran before dispatching and confirm zero remaining instances. Agents self-report "done" but miss uncommon call sites — the grep is the ground truth, not the summary.

**Pitfalls:**
- Subagent self-reports ("created successfully", "38 tests pass") can be wrong — verify with your own terminal call
- Giving vague context ("port the SSRF protection") produces vague output; give exact file paths and invariants
- 3-task concurrent limit means 4+ tasks need two rounds; sequence the rounds so no two tasks that produce files the other needs run in parallel

## The Process

### 1. Read and Parse Plan

Read the plan file. Extract ALL tasks with their full text and context upfront. Create a todo list:

```python
# Read the plan
read_file("docs/plans/feature-plan.md")

# Create todo list with all tasks
todo([
    {"id": "task-1", "content": "Create User model with email field", "status": "pending"},
    {"id": "task-2", "content": "Add password hashing utility", "status": "pending"},
    {"id": "task-3", "content": "Create login endpoint", "status": "pending"},
])
```

**Key:** Read the plan ONCE. Extract everything. Don't make subagents read the plan file — provide the full task text directly in context.

### 2. Per-Task Workflow

For EACH task in the plan:

#### Step 1: Dispatch Implementer Subagent

Use `delegate_task` with complete context:

```python
delegate_task(
    goal="Implement Task 1: Create User model with email and password_hash fields",
    context="""
    TASK FROM PLAN:
    - Create: src/models/user.py
    - Add User class with email (str) and password_hash (str) fields
    - Use bcrypt for password hashing
    - Include __repr__ for debugging

    FOLLOW TDD:
    1. Write failing test in tests/models/test_user.py
    2. Run: pytest tests/models/test_user.py -v (verify FAIL)
    3. Write minimal implementation
    4. Run: pytest tests/models/test_user.py -v (verify PASS)
    5. Run: pytest tests/ -q (verify no regressions)
    6. Commit: git add -A && git commit -m "feat: add User model with password hashing"

    PROJECT CONTEXT:
    - Python 3.11, Flask app in src/app.py
    - Existing models in src/models/
    - Tests use pytest, run from project root
    - bcrypt already in requirements.txt
    """,
    toolsets=['terminal', 'file']
)
```

#### Step 2: Dispatch Spec Compliance Reviewer

After the implementer completes, verify against the original spec:

```python
delegate_task(
    goal="Review if implementation matches the spec from the plan",
    context="""
    ORIGINAL TASK SPEC:
    - Create src/models/user.py with User class
    - Fields: email (str), password_hash (str)
    - Use bcrypt for password hashing
    - Include __repr__

    CHECK:
    - [ ] All requirements from spec implemented?
    - [ ] File paths match spec?
    - [ ] Function signatures match spec?
    - [ ] Behavior matches expected?
    - [ ] Nothing extra added (no scope creep)?

    OUTPUT: PASS or list of specific spec gaps to fix.
    """,
    toolsets=['file']
)
```

**If spec issues found:** Fix gaps, then re-run spec review. Continue only when spec-compliant.

#### Step 3: Dispatch Code Quality Reviewer

After spec compliance passes:

```python
delegate_task(
    goal="Review code quality for Task 1 implementation",
    context="""
    FILES TO REVIEW:
    - src/models/user.py
    - tests/models/test_user.py

    CHECK:
    - [ ] Follows project conventions and style?
    - [ ] Proper error handling?
    - [ ] Clear variable/function names?
    - [ ] Adequate test coverage?
    - [ ] No obvious bugs or missed edge cases?
    - [ ] No security issues?

    OUTPUT FORMAT:
    - Critical Issues: [must fix before proceeding]
    - Important Issues: [should fix]
    - Minor Issues: [optional]
    - Verdict: APPROVED or REQUEST_CHANGES
    """,
    toolsets=['file']
)
```

**If quality issues found:** Fix issues, re-review. Continue only when approved.

#### Step 4: Mark Complete

```python
todo([{"id": "task-1", "content": "Create User model with email field", "status": "completed"}], merge=True)
```

### 3. Final Review

After ALL tasks are complete, dispatch a final integration reviewer:

```python
delegate_task(
    goal="Review the entire implementation for consistency and integration issues",
    context="""
    All tasks from the plan are complete. Review the full implementation:
    - Do all components work together?
    - Any inconsistencies between tasks?
    - All tests passing?
    - Ready for merge?
    """,
    toolsets=['terminal', 'file']
)
```

### 4. Verify and Commit

```bash
# Run full test suite
pytest tests/ -q

# Review all changes
git diff --stat

# Final commit if needed
git add -A && git commit -m "feat: complete [feature name] implementation"
```

## Task Granularity

**Each task = 2-5 minutes of focused work.**

**Too big:**
- "Implement user authentication system"

**Right size:**
- "Create User model with email and password fields"
- "Add password hashing function"
- "Create login endpoint"
- "Add JWT token generation"
- "Create registration endpoint"

## MCP Tools and Subagents

**Subagents do NOT inherit MCP tools from the parent session.** MCP servers (Affine, Granola, etc.) are only available to the parent agent — delegate_task subagents get terminal + file toolsets only.

Consequence: any task that requires MCP tool calls (reading/writing Affine docs, fetching Granola notes, querying Jira via MCP) **must be done directly by the parent agent**, not delegated.

Correct pattern for MCP work:
- Use parallel delegate_task ONLY for terminal/file operations (crawling repos, writing JSON to /tmp, running scripts)
- Have the parent agent do all MCP reads/writes sequentially or in tool-call batches
- If a task mix of MCP + file ops is needed, split: subagent writes to /tmp, parent reads it and calls MCP

```
# WRONG — subagent cannot call mcp_affine_read_doc
delegate_task(goal="Read Affine doc X and summarize it", toolsets=["file"])

# RIGHT — parent reads MCP, subagent processes files
content = mcp_affine_export_doc_markdown(docId="X", workspaceId="...")
write_file("/tmp/doc_x.md", content)
delegate_task(goal="Summarize /tmp/doc_x.md", toolsets=["file"])
```

## Red Flags — Never Do These

- Start implementation without a plan
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed critical/important issues
- Dispatch multiple implementation subagents for tasks that touch the same files
- Make subagent read the plan file (provide full text in context instead)
- Skip scene-setting context (subagent needs to understand where the task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance
- Skip review loops (reviewer found issues → implementer fixes → review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is PASS** (wrong order)
- Move to next task while either review has open issues

## Handling Issues

### If Subagent Asks Questions

- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

### If Reviewer Finds Issues

- Implementer subagent (or a new one) fixes them
- Reviewer reviews again
- Repeat until approved
- Don't skip the re-review

### If Subagent Fails a Task

- Dispatch a new fix subagent with specific instructions about what went wrong
- Don't try to fix manually in the controller session (context pollution)

## Efficiency Notes

**Why fresh subagent per task:**
- Prevents context pollution from accumulated state
- Each subagent gets clean, focused context
- No confusion from prior tasks' code or reasoning

**Why two-stage review:**
- Spec review catches under/over-building early
- Quality review ensures the implementation is well-built
- Catches issues before they compound across tasks

**Cost trade-off:**
- More subagent invocations (implementer + 2 reviewers per task)
- But catches issues early (cheaper than debugging compounded problems later)

## Integration with Other Skills

### With writing-plans

This skill EXECUTES plans created by the writing-plans skill:
1. User requirements → writing-plans → implementation plan
2. Implementation plan → subagent-driven-development → working code

### With test-driven-development

Implementer subagents should follow TDD:
1. Write failing test first
2. Implement minimal code
3. Verify test passes
4. Commit

Include TDD instructions in every implementer context.

### With requesting-code-review

The two-stage review process IS the code review. For final integration review, use the requesting-code-review skill's review dimensions.

### With systematic-debugging

If a subagent encounters bugs during implementation:
1. Follow systematic-debugging process
2. Find root cause before fixing
3. Write regression test
4. Resume implementation

## Example Workflow

```
[Read plan: docs/plans/auth-feature.md]
[Create todo list with 5 tasks]

--- Task 1: Create User model ---
[Dispatch implementer subagent]
  Implementer: "Should email be unique?"
  You: "Yes, email must be unique"
  Implementer: Implemented, 3/3 tests passing, committed.

[Dispatch spec reviewer]
  Spec reviewer: ✅ PASS — all requirements met

[Dispatch quality reviewer]
  Quality reviewer: ✅ APPROVED — clean code, good tests

[Mark Task 1 complete]

--- Task 2: Password hashing ---
[Dispatch implementer subagent]
  Implementer: No questions, implemented, 5/5 tests passing.

[Dispatch spec reviewer]
  Spec reviewer: ❌ Missing: password strength validation (spec says "min 8 chars")

[Implementer fixes]
  Implementer: Added validation, 7/7 tests passing.

[Dispatch spec reviewer again]
  Spec reviewer: ✅ PASS

[Dispatch quality reviewer]
  Quality reviewer: Important: Magic number 8, extract to constant
  Implementer: Extracted MIN_PASSWORD_LENGTH constant
  Quality reviewer: ✅ APPROVED

[Mark Task 2 complete]

... (continue for all tasks)

[After all tasks: dispatch final integration reviewer]
[Run full test suite: all passing]
[Done!]
```

## Remember

```
Fresh subagent per task
Two-stage review every time
Spec compliance FIRST
Code quality SECOND
Never skip reviews
Catch issues early
```

**Quality is not an accident. It's the result of systematic process.**

## Further reading (load when relevant)

When the orchestration involves significant context usage, long review loops, or complex validation checkpoints, load these references for the specific discipline:

- **`references/context-budget-discipline.md`** — Four-tier context degradation model (PEAK / GOOD / DEGRADING / POOR), read-depth rules that scale with context window size, and early warning signs of silent degradation. Load when a run will clearly consume significant context (multi-phase plans, many subagents, large artifacts).
- **`references/gates-taxonomy.md`** — The four canonical gate types (Pre-flight, Revision, Escalation, Abort) with behavior, recovery, and examples. Load when designing or reviewing any workflow that has validation checkpoints — use the vocabulary explicitly so each gate has defined entry, failure behavior, and resumption rules.

Both references adapted from gsd-build/get-shit-done (MIT © 2025 Lex Christopherson).