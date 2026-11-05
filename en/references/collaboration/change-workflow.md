# Change workflow

## Before you start

### Understand before changing [Required]

Before refactoring or a larger change, explain the existing logic, look at the related table schemas, and discuss the details. Start only once the direction is clear.

**Why:** If the direction is wrong, the faster you change, the more rework you create.

**Exceptions:** When the user explicitly says "just build it, no proposal needed", skip the proposal.

**Source:** Team

Before starting, decide where the change goes. For how, see "Choose the best location" in [code-design.md](../general/code-design.md).

## How to proceed

### Move in small steps [Required]

Each change solves one problem. Break cleanup work into a checklist and finish it item by item.

**Why:** Small changes are easy to review and easy to roll back.

**Source:** Team

### Merge branches quickly [Required]

**Applies to:** team project

One branch solves one problem. Open a PR as soon as it is done. The user or the review process decides on merging. Delete the branch after merging.

**Source:** Team

### Stay in scope [Required]

Do only what is within the currently agreed scope. The user's statements and project docs define the exact scope. Handle out-of-scope problems under the next rule.

**Source:** Team

### Handle problems you find [Required]

Do not let unused code or small problems you find along the way go as "keep it for now". Without user authorization, list them in your report and ask whether to handle them. Once authorized, use a separate branch and PR, and do not mix them into the current problem. For areas the project explicitly forbids changing, report only and do not handle.

**Why:** This clears the problem and keeps each change to a single purpose.

**Source:** Team

For general problems, research mature solutions first. See "Prefer mature libraries for general problems" in [dependencies.md](../general/dependencies.md).

## Verification

### Keep verification fast [Required]

Locally, run only lint, type-check, and the tests related to the change. Do not run the full gates or the full test suite. CI is the check before merging.

**Why:** Full verification takes a long time, and most of it is unrelated to the current change.

**Exceptions:** When a change has global impact, such as shared modules or build configuration, widen the verification scope and tell the user. When the project has no CI or full verification is fast, run everything locally.

**Source:** Team

For measurement requirements on performance changes, see [performance.md](../general/performance.md).

### Report verification results truthfully [Required]

State what you ran, what happened, what was not verified, what is unfinished, and what needs the user's decision. Do not report anything you did not run as passed.
