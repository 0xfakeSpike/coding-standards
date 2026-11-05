# General coding principles

This document contains only principles that do not depend on language or domain. The team's overall requirement: keep code simple through architectural design, not after-the-fact patching. Solve problems at the root, and delete what is not used.

## How to solve problems

### Do not patch [Required]

Do not develop in awkward ways or fix bugs with temporary patches. Find the root cause and fix it at the root.

**Why:** Patches hide the real problem and make the next change harder.

**Example:** A field is occasionally empty. Avoid just adding a null check where it is read. Prefer finding out why it is empty and fixing it at the write or in the data flow.

**Source:** Team

### Think about the whole system [Required]

Changes must fit the system's layering and division of responsibilities. Do not focus only on the bug or feature at hand.

**Source:** Team

### Do not add complexity for edge cases [Required]

For rare scenarios with acceptable consequences, do not add a mechanism to handle them if the benefit does not justify the long-term maintenance cost. Assess cases involving money or data loss separately.

**Example:** Adding a fallback path or scheduled compensation for failures that almost never happen is usually not worth it.

**Source:** Team

### Do not build for hypothetical needs [Recommended]

Build only what is needed now. Do not reserve extension points that have no consumer.

**Why:** Elements added early carry ongoing maintenance cost, and often go unused once needs change.

**Exceptions:** Investments that make code easy to change, such as refactoring, automated tests, and continuous delivery, are not limited by this rule.

**Source:** Fowler YAGNI, Google Code Review Guide

## Simple design

Express each piece of knowledge once. See "Keep one copy of each thing" in [responsibilities.md](responsibilities.md).

### Judge simplicity in priority order [Recommended]

Check in order: passes the tests, expresses intent, has no duplication, has the fewest elements. Delete anything that does not serve the first three.

**Source:** Beck: Rules of Simple Design

### Put readability first [Recommended]

Code is read far more often than it is written. Clever writing must not come at the cost of readability.

**Source:** Beck: Rules of Simple Design

### Minimize surprise [Recommended]

Functions and modules must behave in line with their names and common expectations. Query operations have no hidden side effects.

## Functions

### One function, one thing [Recommended]

If describing it takes the word "and", it usually should be split.

### Use early returns to reduce nesting [Recommended]

Handle exceptional branches first and keep the main flow at the outermost level. Do not nest more than 3 levels.

### Make argument meaning readable at the call site [Recommended]

Avoid too many parameters, and avoid boolean flag parameters with unclear meaning. Specific techniques vary by language. See the language layer.

### Limit function size with tools [Recommended]

Project gates set the thresholds for function length and complexity. Do not rely on manual estimates.

## Files

### Split files by responsibility [Recommended]

A growing file usually means mixed responsibilities. Split by responsibility, not mechanically by line count. Project gates set the length threshold. Generated code does not count.

## Magic values

### Name values with business meaning [Recommended]

Extract status codes, thresholds, and type values into named constants or enums, and keep them in a single place.

## Comments

### Comments record only what the code cannot say [Required]

Do not write comments that explain what the code does. When code needs comments to be understood, improve naming and structure first. Do not clean up a project's existing comments just for this rule. Handle outdated ones when you change the related code.

When the code cannot express a reason, write it in one line next to the code, such as an external system limit, counter-intuitive behavior, or a single-point trade-off like ignoring an error, a type assertion, or suppressing lint.

**Why:** Comments that restate the code easily drift away from it. A reason written next to the code is what the person changing it will actually see.

**Example:** "The payment gateway rejects the same order number for 60 seconds, so do not retry immediately" is a reason, so write it. "Loop over the order list" restates the code, so do not.

**Exceptions:**

- Directives read by tools are not comments, such as `@ts-expect-error`, `# type: ignore`, `//go:build`, and lint ignore directives.
- License headers.
- API docs for published libraries, which consumers rely on to understand the API.

**Tooling:** If a lint rule requires doc comments, turn it off, except for published libraries.

**Source:** Team

### Put notes in project docs [Required]

Record business background, architectural trade-offs, and overall external system constraints that span several places in the code in project docs. See [project-docs.md](project-docs.md). A reason that concerns only one place in the code goes next to that code.

**Source:** Team

### Do not keep commented-out code [Recommended]

Version control takes care of history.

### Keep to-dos out of code [Recommended]

Record them in tickets or project docs.
