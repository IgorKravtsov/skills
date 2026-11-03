# Skills

Agent skills for Claude Code, Codex, Cursor, OpenCode and other agents, installable with [`npx skills`](https://skills.sh).

| Skill | What it does |
| --- | --- |
| `t3-browser-test` | Spawns an independent agent via T3 Code orchestration to test current changes in T3's real browser preview. |
| `t3-critical-review` | Spawns a second agent via T3 Code orchestration to critically review your changes and iterates until every finding is resolved. |
| `babysit-pr` | Opens a PR and babysits it through bot/AI review (CodeRabbit, Sonar, Copilot, …): fixes or pushes back on every comment in its own thread until each reviewer is done or broken. |
| `notebooklm` | Operates Gemini Notebook through the `notebooklm-py` CLI / Python API (skill shipped with notebooklm-py v0.8.5). |

## Install

All skills from this repo:

```bash
npx skills add IgorKravtsov/skills --all
```

Only for Claude Code, globally:

```bash
npx skills add IgorKravtsov/skills --skill '*' -a claude-code -g -y
```

A single skill:

```bash
npx skills add IgorKravtsov/skills --skill t3-critical-review
```

Update later with `npx skills update`.

## Recommended third-party skills

```bash
# Matt Pocock's engineering / productivity skills
npx skills add mattpocock/skills -y \
  --skill grill-me --skill grill-with-docs --skill grilling \
  --skill codebase-design --skill improve-codebase-architecture \
  --skill prototype --skill research --skill tdd --skill teach \
  --skill wayfinder --skill handoff --skill wait-what

# Browser automation (Vercel)
npx skills add vercel-labs/agent-browser --skill agent-browser -y

# Ponytail (lazy senior dev mode)
npx skills add dietrichgebert/ponytail --skill ponytail -y
```
