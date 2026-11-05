# Git workflow

## Branches

### Branch from the latest main branch [Required]

**Applies to:** team project

Start new work from the latest main branch. Do not continue developing on a merged branch.

**Source:** Team

### One branch, one problem [Required]

**Applies to:** team project

See "Merge branches quickly" in [change-workflow.md](../collaboration/change-workflow.md).

**Source:** Team

### Keep branches short-lived [Recommended]

Use short-lived branches only for review and CI, and merge them back to trunk quickly. Hide unfinished features behind feature flags instead of relying on long-lived branches.

**Source:** Trunk Based Development, DORA: Continuous Delivery

### Branch naming [Recommended]

Use `type/summary`. Include the ticket number when there is one.

```text
feat/PROJ-123-order-export
fix/login-redirect-loop
```

### Protect the main branch [Required]

**Applies to:** team project

Do not commit directly to the main branch. Merge through PRs, and enable CI checks.

### Do not force push to shared branches [Required]

It overwrites other people's commits. When you need to rewrite history on a personal branch, use `--force-with-lease`.

## Commits

### Use Conventional Commits [Recommended]

```text
<type>(<scope>): <description>

<body>

<footer>
```

| type | Meaning |
|---|---|
| `feat` | New feature, required by the spec |
| `fix` | Defect fix, required by the spec |
| `refactor` | Refactoring that does not change behavior |
| `perf` | Performance optimization |
| `test` | Tests |
| `docs` | Docs |
| `build`, `ci` | Build and CI |
| `chore` | Other miscellaneous work |

Format requirements:

- A type may be followed by a scope in parentheses, then a colon and a space.
- Leave a blank line between the description and the body.
- Leave a blank line between the body and the footer. Tokens use `-` in place of spaces, such as `Reviewed-by`.

**Tooling:** commitlint

**Source:** Conventional Commits

### Say what changed in the description [Recommended]

`fix bug`, `update`, and `phase 1` are all unacceptable. The first line is a short, complete description. Add the reason and background in the body when needed.

**Source:** Google Code Review Guide

### One commit, one thing [Recommended]

**Applies to:** team project

Unrelated cleanup and refactoring go on a separate branch and PR, which makes review and rollback easier. Use separate commits only to split work within the same problem.

**Source:** Google Code Review Guide

## Commit contents

### Do not commit sensitive content or build output [Required]

This includes secrets, local environment configuration, build output, dependency directories, personal IDE configuration, and large binaries. The project decides as a whole whether to commit generated code.

### Rotate leaked secrets immediately [Required]

Deleting the file or rewriting history is not enough. The secret may already have been pulled or cached.

## Versions

### Use semantic versioning [Recommended]

The format is `X.Y.Z`. Once tagged, a version's contents do not change. Changes require a new release. `0.y.z` means initial development, and the public API is not considered stable.

**Source:** SemVer

## Merge strategy

### Use one merge strategy per project [Recommended]

Choose one of squash, merge, or rebase. Squash is recommended, so each commit on the main branch maps to one complete change.
