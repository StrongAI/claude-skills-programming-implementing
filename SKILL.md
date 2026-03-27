---
name: implement
user-invocable: false
description: Internal skill invoked by /programming chain. Use when implementing a single task from a plan. Defines the TDD implementation loop — write tests, implement, validate tests, audit code, repeat if necessary. Each dispatched subagent follows this workflow. Triggers on task execution within a dispatch-programmers context.
---

# Implement

The per-task implementation loop. You have a single task from a plan. Follow this loop until the task is done.

## Prerequisites

Before starting, you MUST have:
- A task description from the plan (what to build, acceptance criteria)
- The appropriate language/framework skill loaded (`/programming-swift`, `/programming-typescript`, etc.)

## The Loop

```
┌─────────────────────────────────────────┐
│ 1. Write Tests                          │
│    Tests for the acceptance criteria    │
│    They MUST fail (red)                 │
├─────────────────────────────────────────┤
│ 2. Implement                            │
│    Write the minimum code to pass       │
│    Follow the language skill patterns   │
├─────────────────────────────────────────┤
│ 3. Validate Tests                       │
│    Run tests — they MUST pass (green)   │
│    If they fail, go back to step 2      │
├─────────────────────────────────────────┤
│ 4. Audit (SEPARATE AGENT)              │
│    Launch /audit-code as new agent      │
│    Reads plan + changed files           │
│    Returns findings report              │
├─────────────────────────────────────────┤
│ 5. Audit clean?                         │
│    YES → commit, done                   │
│    NO  → back to step 1 with findings   │
└─────────────────────────────────────────┘
```

### Step 1: Write Tests

Write tests FIRST. Before any implementation code.

- One test per acceptance criterion from the task
- Include edge cases and error paths from the task description
- Run the tests — they must ALL FAIL. If any pass, something is wrong (pre-existing implementation, wrong test, wrong assertion)
- Use the testing framework specified by the language skill (`@Test` for Swift, `jest`/`vitest` for TypeScript, etc.)

### Step 2: Implement

Write the minimum code to make the tests pass.

- Follow the patterns and conventions from the loaded language skill
- Do NOT add behavior that isn't covered by a test
- Do NOT refactor yet — just make it work

### Step 3: Validate Tests

Run all tests (not just the new ones).

- New tests must pass
- Existing tests must not break
- If tests fail, fix the implementation (not the tests) and re-run
- If an existing test breaks, understand why before changing anything

### Step 4: Audit (Separate Agent)

Launch a **separate audit agent** using `/audit-code`. This is NOT a self-review — it is an independent agent with no implementation context.

The audit agent:
- Reads the task requirements from the plan
- Reads every file you changed
- Reads prior audit handoff messages from statement-mcp (if any previous rounds exist)
- Produces a findings report (verified, missing, deviated, unplanned, dead paths, untested)
- **Writes a handoff message to statement-mcp** with its findings for the implementer — what to fix, what's missing, what deviated. The implementer reads this at the start of the next round instead of relying on inline context.

You MUST wait for the audit to complete before proceeding.

### Step 5: Audit Clean?

**If the auditor found issues:**
- Go back to step 1 with the audit findings as new requirements
- Write tests for the gaps the auditor identified
- Implement the fixes
- Validate tests
- Launch a NEW audit agent (step 4 again)
- Repeat this loop until the auditor returns a clean report

**If the auditor found nothing:**
- Commit the task
- Report completion to the dispatcher

The loop does not end until the auditor says it's clean. There is no shortcut.

## Statement-MCP Handoffs

All communication between implementation and audit rounds goes through statement-mcp, not inline context.

**Implementer writes** after each round:
- What was implemented or fixed
- Which tests were added
- Which audit findings were addressed
- Any decisions made and why

**Auditor writes** after each round:
- Findings report (what passed, what failed, what's missing)
- Specific instructions for the next implementation round
- Whether prior audit findings were properly resolved

This creates a durable trail. Each new agent reads the handoff messages to understand where the loop stands without needing the prior agent's context.

## Rules

- Tests come FIRST. No exceptions. No "I'll add tests after."
- The implementation serves the tests, not the other way around. Never weaken a test to make implementation easier.
- One task, one loop. Do not combine multiple plan tasks into one implementation pass.
- Stay within scope. If you discover work that should be done but isn't in your task, note it for the dispatcher — do not do it.
- Use the debugger for failures, not print statements or reading code and guessing.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing implementation before tests | Always step 1 first |
| Tests that test implementation details | Test behavior and contracts, not internals |
| Skipping the audit step | Step 4 catches what steps 1-3 miss |
| "Tests pass so I'm done" | Passing tests ≠ complete implementation |
| Fixing a test to match wrong behavior | Fix the code, not the test |
| Adding unrequested features | Stay within task scope |
| Refactoring during implementation | Make it work first, refactor after audit passes |
