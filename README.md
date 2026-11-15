# coding-standards

> Now it's the AI's turn to suffer.

A Claude Code skill that makes your agent write and review code by an opinionated set of engineering standards. The rules come in Chinese and English.

一个 Claude Code skill，让 agent 按一套有立场的工程规范写代码和审查代码。规则有中文和英文两个版本。

## What it believes

- Solve problems at the root. No patches.
- Simple code comes from good architecture, not cleanup afterwards.
- Delete what nobody uses.
- Keep one copy of each thing.
- No wrappers without a purpose.
- No extra machinery for rare, harmless failures.
- Comments say why, never what.

## Install

The rules come in two editions with the same layout. Install one of them.

| Edition | Plugin |
|---|---|
| Chinese | `coding-standards-cn` |
| English | `coding-standards-en` |

In Claude Code, add this repository as a marketplace, then install the edition you want:

```text
/plugin marketplace add 0xfakeSpike/coding-standards
```

```text
/plugin install coding-standards-cn@coding-standards
```

Marketplaces like this one do not update automatically. To get new rules, run these from your terminal:

```bash
claude plugin marketplace update coding-standards
```

```bash
claude plugin update coding-standards-cn@coding-standards
```

Start a new session afterwards. The skill loads when the agent writes, changes, debugs or reviews code.

安装后新开一个会话。agent 写代码、改代码、排查问题或审查代码时会自动使用。

## What's inside

Each edition has the same files. Paths below are relative to `cn/skills/coding-standards/` or `en/skills/coding-standards/`.

| Layer | Path | Covers |
|---|---|---|
| Collaboration | `references/collaboration/` | How the agent works and reports |
| Languages | `references/languages/` | TypeScript, Python, Go |
| Frontend | `references/frontend/` | Components, React, Vue, Next.js, Nuxt, CSS, state, performance, accessibility |
| Backend | `references/backend/` | APIs, architecture, reliability, observability, data, async, security, AI apps |
| General | `references/general/` | Design, naming, errors, refactoring, testing, Git, CI/CD, containers, releases |

The agent reads `SKILL.md` first, then only the files it needs.

## How rules work

Each rule is either 【必须】 ([Required]) or 【推荐】 ([Recommended]). Some rules only apply to certain projects:

| 适用 | Applies to | Meaning |
|---|---|---|
| 多人协作 | team project | Multiple developers with PRs and CI |
| 线上服务 | production service | Production systems with real users |
| 高风险 | high risk | Money, personal data, or direct public exposure |

When rules conflict, the order is:

```text
user instructions > project conventions > team rules > language > domain > general
```

Security baselines in `SKILL.md` are not overridden by project conventions.

## Use it as a quality gate

The skill guides the agent, but on its own it does not block anything. To make it a gate on pull requests:

1. **Check what tools can check first.** Secrets, ignored errors, SQL built from strings, missing timeouts and formatting are better caught by linters, formatters and scanners such as gitleaks, golangci-lint, ruff, ESLint and semgrep. They are stable and cheap. The AI review covers what they cannot, such as design, responsibilities and needless wrappers.
2. **Declare the project's scenarios.** Say in the project's `CLAUDE.md` which of team project, production service and high risk apply. In gate mode the agent does not guess them, and without the declaration no rule with an `**Applies to:**` field can block.
3. **Run the review in CI.** Copy [examples/standards-review.yml](examples/standards-review.yml) to `.github/workflows/`, replace `make check` with your project's own checks, and add an `ANTHROPIC_API_KEY` secret. The `checks` job runs your tests and linters. The `review` job then asks for a review in gate mode, which ends with a `GATE: PASS`, `GATE: FAIL` or `GATE: ERROR` line, and fails unless the result is PASS. Only Must fix items block.
4. **Make it required.** Mark both jobs as required status checks in branch protection. Start with `continue-on-error: true` on the review job for a few weeks, check the false positives, then remove it.

The review job cannot run tests itself. Tests execute the PR's code, and in that job the code could read the API key. So the checks run in their own job without secrets, and the review reads their output from `checks.log`. A candidate that needs running code to confirm and that the output does not settle goes under Needs confirmation, which does not block.

An AI review is not deterministic, and the code it reviews can try to talk it into passing. Keep security checks in the tools from step 1, and give the review job no write access or extra secrets. The example also loads only user settings (`--setting-sources user`, `--strict-mcp-config`), so a PR cannot add hooks or MCP servers to the review through `.claude/` files.

要把它当作 PR 的质量门禁：先用工具检查能自动查的问题，在项目 `CLAUDE.md` 里声明适用场景，再把示例 workflow 加进 CI 并设为必需检查。示例里项目自己的检查在单独的 job 里先跑，审查只读它的输出。

## Make it yours

The rules marked `**来源：** 团队` (`**Source:** Team` in English) are one team's opinions, such as keeping comments to the why. Fork it and change what doesn't fit. A project's own documented conventions take priority anyway.

These rules are for writing better code. They do not replace your own security, privacy or legal review.

## Sources

Public references are listed in `SOURCES.md` ([Chinese](cn/skills/coding-standards/references/SOURCES.md), [English](en/skills/coding-standards/references/SOURCES.md)). Rules are distilled and rewritten, not copied. The referenced materials remain under their own licenses.

## License

[MIT](LICENSE)
