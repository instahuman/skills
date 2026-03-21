# InstaHuman Skills

Agent skills for [InstaHuman](https://instahuman.com) — get structured human feedback from real people via MCP.

## Install

```bash
npx skills add instahuman/skills
```

This installs skills into your coding agent (Claude Code, Codex, Cursor, Gemini CLI, and [40+ more](https://www.npmjs.com/package/skills#supported-agents)).

### Install specific skills

```bash
# Core only (create_job, wait_job, get_job, list_jobs)
npx skills add instahuman/skills --skill instahuman

# All skills
npx skills add instahuman/skills --skill '*'
```

## Available Skills

| Skill | Description |
|---|---|
| **instahuman** | Core tools: create jobs, wait for results, inspect progress |
| **instahuman-lifecycle** | Pause, resume, cancel, stop jobs, search testers, manage projects |
| **instahuman-disputes** | Close jobs, approve payouts, submit disputes, check settlement |

## Requirements

You need an InstaHuman API key (`ih_...`). Get one at [instahuman.com](https://instahuman.com).

## Links

- [InstaHuman](https://instahuman.com)
- [MCP endpoint documentation](https://instahuman.com/SKILL-full.md)
