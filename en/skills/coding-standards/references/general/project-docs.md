# Project docs

Comments only record single-point reasons, so background and notes that span several places in the code go in project docs.

## Notes

### Record what code cannot express [Required]

This includes business background, architectural trade-offs, and overall external system constraints. A reason that concerns only one place in the code goes next to that code. See [principles.md](principles.md). The project decides where it goes and how it is organized. State clearly what to watch out for and why.

**Source:** Team

### Update docs with the code [Recommended]

Update docs when you change behavior. Delete outdated content.

### Record to-dos in tickets [Recommended]

When there is no ticket system, keep them together in project docs.

## External docs

### Use professional, concise wording [Required]

Do not use colloquial language.

**Source:** Team

## README

### Make the README answer getting-started questions [Recommended]

| Content | Notes |
|---|---|
| Project purpose | One or two sentences |
| Requirements | Language, tools, and versions |
| How to start | Commands that can be run directly |
| Common commands | Test, lint, build |
| Directory structure | Responsibilities of the main modules |
| Docs entry point | Where other docs are |

## Changelog

### Follow Keep a Changelog when needed [Recommended]

- Put the latest version first. State the date of each version, using ISO 8601.
- Maintain an `Unreleased` section.
- Group by Added, Changed, Deprecated, Removed, Fixed, Security.

**Source:** Keep a Changelog

## Architecture decisions

### Write an ADR for important decisions [Recommended]

One ADR records one decision with its rationale, trade-offs, and consequences.

```text
docs/adr/adr-001-short-title.md

# Title
## Status
## Context
## Decision
## Consequences
```

**Source:** Nygard ADR, ADR Site

### Number ADRs incrementally and never reuse numbers [Recommended]

When a decision is overturned, keep the old record and mark it as superseded. ADRs are decision history and do not count as outdated docs.

**Source:** Nygard ADR

## Technical proposals

### State the core problem first in a proposal [Recommended]

Include background, goals, non-goals, the proposal, trade-offs, risks, and how to verify.

When the user says "just build it, no proposal needed", do not write a proposal.

**Source:** Team

## Runbooks

### Provide runbooks for production services [Recommended]

Cover deployment, key metrics, common failures, and the steps to handle them.
