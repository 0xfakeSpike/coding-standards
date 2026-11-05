# Refactoring

### Fix small problems too [Required]

Handle a small problem when you find it. Do not let it go because it "does not affect runtime". For problems outside the current task's scope, ask first as described in [change-workflow.md](../collaboration/change-workflow.md).

**Source:** Team

### Separate refactoring from feature changes [Recommended]

**Applies to:** team project

Refactoring does not change behavior. Put it in a separate branch and PR.

**Why:** When they are mixed, reviewers cannot tell whether a behavior change is intentional.

**Source:** Google Code Review Guide

### Refactor in small steps [Required]

Keep the system working at every step. Solve one problem at a time.

**Source:** Team

### Start with structure [Recommended]

First check whether the division of responsibilities and the data model are sound, then improve how functions are written. If structural problems stay, local cleanup soon stops helping.

### Protect behavior when there are no tests [Recommended]

Before changing critical logic that lacks tests, write tests that record the current behavior.

For outdated tests, see "Delete outdated tests" in [testing-strategy.md](testing-strategy.md).

## Technical debt

By default, handle technical debt directly when you find it. For how, see "Handle problems you find" in [change-workflow.md](../collaboration/change-workflow.md).

### Record debt you leave behind [Recommended]

When it truly cannot be handled right away, record it as described in [Record to-dos in tickets](project-docs.md). State the problem, the impact, and the conditions for handling it. Do not put it in code comments.

## Code smells

| Smell | Common fix |
|---|---|
| Duplicated logic | Consolidate into one copy |
| Overlong function | Extract by responsibility |
| Too many parameters | Introduce a meaningful data structure |
| Function that only forwards a call | Inline and delete |
| Condition that always takes the same branch | Delete the dead branch |
| Configuration no longer used | Delete |
| Code that takes explaining to understand | Improve naming and structure |
