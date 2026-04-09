---
name: refactoring-guidance
description: |
  WHAT: Language-agnostic corrective guidance for the refactoring phase.
  WHEN: Agent is restructuring code, fixing code smells, reducing complexity, or improving maintainability.
  NOT FOR: Writing new features, debugging runtime errors, performance tuning, or object design decisions.
allowed-tools: [Read, Grep, Glob]
---

# Refactoring Guidance

You know Fowler's catalog and standard code smells. These directives correct where that knowledge is most often misapplied during code generation.

## Scope: Fix What You Were Asked to Fix

- Refactor only the targeted code. Resist "while I'm here" on surrounding code.
- If you spot other smells, note them — don't fix them in the same change.
- A refactoring PR that touches one smell is reviewable. One that touches five is not.

## Prioritize by Impact, Not by Textbook Severity

Not all smells deserve fixing. Rank by real-world cost:

- **Fix now**: Code that causes bugs across the team, blocks feature work, or sits in a file that appears in >30% of recent commits.
- **Fix soon**: Duplication across 3+ call sites, functions that require 20+ lines of test setup, logic scattered across 4+ files for one concept.
- **Leave alone**: One-off long parameter lists, single naming nitpicks, style inconsistencies in stable code nobody edits.

A working codebase the team understands beats a theoretically perfect one.

## Default Simple, Escalate With Evidence

Apply the simplest pattern that resolves the smell. Escalate only when specific criteria are met:

- **Extract function** is the default. Escalate to a method object or dedicated class only when: 4+ interacting local variables, would need 3+ params passed between extracted pieces, or the logic warrants its own test suite.
- **Keep logic inline** is the default. Extract to a separate module/class only when: 2+ call sites need it, it has its own state, or it's independently testable and worth testing alone.
- **A simple conditional is fine.** Escalate to polymorphism or pattern matching only when the same type/status dispatch appears in 3+ places.

If you can't articulate why the heavier pattern is necessary, use the lighter one.

## Moving Code Is Not Simplifying Code

- Extracting to a new file doesn't reduce complexity — it just relocates it. The result must be simpler to understand, not just shorter in each file.
- A shared abstraction with one call site is premature. Inline it.
- Wrapping a function in a class without adding state or encapsulation is ceremony, not design.

## One Structural Change at a Time

- Each commit changes one thing structurally. Rename, then extract, then move — not all at once.
- Never change behavior while restructuring. If you need both, separate commits.
- Run the full relevant test suite after each step, not just at the end.

## Tests Gate Everything

- If tests don't cover the code path, write characterization tests FIRST that capture current behavior. Then refactor.
- If you can't verify the refactoring preserved behavior, you're guessing.
- New tests assert existing behavior before the change. New behavior is a separate concern.

## Know When to Stop

- The goal is "measurably better," not "perfect." Stop when complexity is reduced and the code is easier to change.
- If a refactoring introduces as much indirection as it removes duplication, it's a lateral move. Revert it.
- Ask: would a new team member find this easier to understand? If the answer is ambiguous, the refactoring isn't worth it.
