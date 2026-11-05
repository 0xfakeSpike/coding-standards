---
name: coding-standards
description: Team coding standards handbook. Use before writing, changing, refactoring, debugging, or reviewing code, and when designing APIs and table schemas, writing tests, committing, and reviewing changes. Not for tasks unrelated to development.
---

# Coding Standards

Detailed rules live under `references/`. Read the `index.md` of the relevant layer first, then pick files.

**Limit how much you read:** a task usually needs 3 to 6 rule files, chosen because they relate directly to the current change. Do not read every file in an index for completeness. Start work once you have read enough. A topic you have not read does not mean there is no rule for it. Read it when a related question comes up.

| Layer | Question it answers | Entry point |
|---|---|---|
| Collaboration layer | How to carry out work and report on it | [collaboration/index.md](references/collaboration/index.md) |
| Language layer | How to write this language | [languages/index.md](references/languages/index.md) |
| Frontend layer | How to design frontend systems | [frontend/index.md](references/frontend/index.md) |
| Backend layer | How to design backend systems | [backend/index.md](references/backend/index.md) |
| General layer | Principles and engineering practices independent of language and domain | [general/index.md](references/general/index.md) |

Rule sources are listed in [SOURCES.md](references/SOURCES.md).

## Workflow

1. **Check the project's existing conventions first.** Look at lint and formatter configuration, `CLAUDE.md`, `CONTRIBUTING.md`, project docs, and the surrounding code. Only what configuration and docs state explicitly counts as a convention. Surrounding code decides style only and does not vouch for its own defects.
2. **Read the collaboration rules.** Read `collaboration/change-workflow.md` before changing code. Skip it when you are only reviewing or answering a question.
3. **Identify the language and domain, then read the relevant files.** One task may involve several files. For example, adding an endpoint usually means reading API design, backend architecture, and database rules.
4. **Read the general layer as needed.** Read it when the work involves design judgment, deletion, debugging, committing, or releasing.
5. **Explain deviations.** When you do not follow a [Required] rule, tell the user.

Rules have two levels:

- **[Required]**: violating it directly causes defects, security problems, or data loss, or it is a practice the team explicitly requires. Follow it unless there is a clear reason not to.
- **[Recommended]**: the default. Adjust it when a better option exists. Style, tooling gates, process, architectural preferences, and pure performance tuning belong at this level unless the team explicitly requires them.

Some rules or files carry an `**Applies to:**` field and take effect only in the matching scenario. When a project is not in that scenario, the rule does not apply and no deviation needs explaining.

| Applies to | Meaning |
|---|---|
| team project | Several developers, with PRs and CI |
| production service | A live system serving real users, with availability requirements |
| high risk | Involves money or personal data, or is directly exposed to the public internet |

This handbook covers only how to write code. It does not cover business decisions, team management processes, tool version notes, or language basics. Those are decided by the project, the user, and official documentation.

## Priority when rules conflict

From highest to lowest: **explicit user instructions > existing project conventions > team rules > language layer > domain layer > public standards in the general layer.**

Team rules are rules whose Source field includes "Team". The language layer ranks above the domain layer because domain rules describe goals, and the implementation has to fit the language's idioms.

Security rules are the exception: the baseline rules below are not overridden by project conventions, and a [Recommended] rule at a higher level cannot override a security [Required] rule at a lower level. When the user explicitly asks to deviate, explain the risk and then proceed.

## Baseline rules

Follow these rules without needing to read any reference.

All code:

- Fix problems at the root, not with temporary patches. During an incident you may contain the damage first, but a root fix is still required afterwards.
- Put each change where the responsibility best fits.
- Once you confirm nothing uses it, delete unused code and content. When it is unrelated to the current task, list it in your report and ask first, then handle it on a separate branch once authorized. Ask the user first about production data and anything unrecoverable.
- Keep one copy of each thing.
- Do not add wrappers without a purpose.
- Do not add fallback paths or scheduled compensation for rare failures with acceptable consequences, and do not add features the user did not ask for. Assess cases involving money or data loss separately, and tell the user first when one is truly needed.
- Comments record only reasons the code cannot express, such as external system limits, counter-intuitive behavior, and single-point trade-offs, not what the code does. Put notes that span several places in the code in project docs. Exceptions are license headers and API docs for public libraries.
- Do not commit keys, passwords, tokens, or real user data.
- Do not ignore errors. Handle them, or pass them up with context.
- Report verification results truthfully: what you ran, what happened, and what was not verified.

Frontend:

- Do not write unsanitized content into HTML.
- Do not put secrets in frontend code or public environment variables.
- The backend is the authority on permissions.

Backend:

- Validate all external input at the boundary. Parameterize SQL values, and use fixed mappings for structure such as table and column names.
- Endpoints require login and data permission checks by default. Declare public endpoints explicitly.
- External calls must set a timeout.
- Logs must not contain sensitive information such as passwords, tokens, or full ID numbers.
- Keep business logic out of the database.

## Review output format

When the user asks for a review against these standards, give the conclusion first, then list findings from most to least severe:

```markdown
## Review conclusion
A one-sentence summary, with the number of must-fix and should-fix items.

## Must fix
1. `path:line`: description of the problem
   - Rule: category / rule name
   - Impact: what happens if it is not fixed
   - Fix: the specific change, with code if needed

## Should fix
(Same format as above)

## Done well
(Optional. List only practices worth keeping.)
```

Report only real problems. If a formatter can fix a formatting issue automatically, merge those issues into one item and recommend adopting the tool instead of listing them line by line.
