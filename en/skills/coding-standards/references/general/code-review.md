# Code review

## Author

**Applies to:** team project (all rules in this section)

### Keep each PR to one self-contained change [Recommended]

About 100 lines is a reasonable size. About 1000 lines is usually too large. File count also counts toward size: a change spread across many files is just as hard to review.

**Exceptions:** Deletions can be larger. Reviewing the deletion of a whole file costs very little.

**Source:** Google Code Review Guide

### Keep tests and logic in the same PR [Recommended]

For splitting refactoring from features, see [refactoring.md](refactoring.md). When adding an API, include real usage of it in the same PR.

**Source:** Google Code Review Guide

### Describe the change and the reason [Recommended]

Include what changed, why, how it was verified, and what the risks are.

**Source:** Google Code Review Guide

### Read the diff before submitting [Recommended]

Check for debug code, unrelated changes, and missing tests.

### Put explanations in code or docs [Required]

Anything that needs explaining during review must lead to better code or go into project docs. Do not leave it only in the review tool.

**Source:** Team, Google Code Review Guide

## Reviewer

### Approve once code health clearly improves [Recommended]

**Applies to:** team project

Do not chase perfection, but do not merge changes that clearly worsen code health.

**Source:** Google Code Review Guide

### Review in priority order [Recommended]

**Applies to:** team project

1. Design: whether responsibilities and layers are sound, and whether the change is in the best location.
2. Functionality: logic, edge cases, and concurrency.
3. Security: permissions, input, and sensitive data.
4. Complexity: whether it is over-designed, and whether there is unused code.
5. Tests: whether they fail when something goes wrong.
6. Naming and style: do not rely on people for what tools can handle.

**Source:** Google Code Review Guide, Team

### Do a dedicated security review for high-risk changes [Required]

**Applies to:** high risk

When a change involves any of the following, check it item by item against the [backend security rules](../backend/index.md) and [frontend security rules](../frontend/security.md), and state the results in the PR description:

- Authentication and authorization.
- Payments and funds.
- Encryption, decryption, and keys.
- File uploads and file paths.
- Queries, commands, or external URLs built from user input.
- Storage and transmission of sensitive data.

### Ask for clarification when you do not understand [Recommended]

**Applies to:** team project

If the reviewer cannot understand it, future readers will not either.

**Source:** Google Code Review Guide

### Settle style disagreements with the standards [Recommended]

**Applies to:** team project

The standards decide style. Personal preference does not block a merge.

**Source:** Google Code Review Guide
