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

The rules come in two editions with the same layout: `cn/` (Chinese) and `en/` (English). Install one of them, because both use the skill name `coding-standards`.

Clone the repository outside your skills folder:

```bash
git clone https://github.com/0xfakeSpike/coding-standards.git ~/.claude/coding-standards
```

Then link one edition as the skill. For all your projects:

```bash
mkdir -p ~/.claude/skills
```

```bash
ln -s ~/.claude/coding-standards/cn ~/.claude/skills/coding-standards
```

For one project, run this from the project root:

```bash
mkdir -p .claude/skills
```

```bash
ln -s ~/.claude/coding-standards/cn .claude/skills/coding-standards
```

For the English edition, replace `cn` with `en`.

Start a new session afterwards. The skill loads when the agent writes, changes, debugs or reviews code.

安装后新开一个会话。agent 写代码、改代码、排查问题或审查代码时会自动使用。

## What's inside

Each edition has the same files. Paths below are relative to `cn/` or `en/`.

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

## Make it yours

The rules marked `**来源：** 团队` (`**Source:** Team` in English) are one team's opinions, such as keeping comments to the why. Fork it and change what doesn't fit. A project's own documented conventions take priority anyway.

These rules are for writing better code. They do not replace your own security, privacy or legal review.

## Sources

Public references are listed in [cn/references/SOURCES.md](cn/references/SOURCES.md) and [en/references/SOURCES.md](en/references/SOURCES.md). Rules are distilled and rewritten, not copied. The referenced materials remain under their own licenses.

## License

[MIT](LICENSE)
