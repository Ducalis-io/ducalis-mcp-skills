# Ducalis Skills

AI skills for working with [Ducalis](https://ducalis.io) — a B2B SaaS tool for backlog prioritization.

## What's included

### `ducalis-use` skill

Teaches AI agents how to work with Ducalis via `read_ducalis` / `write_ducalis` MCP tools.

- **SKILL.md** — core concepts, universal rules, capabilities index
- **references/** — detailed guides loaded on demand:
  - `priorities.md` — top items, ranking, evaluation awareness
  - `evaluation-write.md` — scoring, voting, batch evaluation
  - `evaluation-context.md` — criteria, scales, how to evaluate
  - `evaluation-discovery.md` — what to evaluate, progress tracking
  - `issue-search.md` — find and filter issues
  - `issue-write.md` — create, update, delete issues
  - `gantt.md` — timeline planning, scheduling, dependencies
  - `questions.md` — question threads, discussions
  - `board-info.md` — board details, settings, members
  - `capacity-planning.md` — sprint capacity planning workflow (example)
  - `compliance-risk.md` — regulatory compliance risk workflow (example)

## Installation

### Claude Code

```bash
claude skills add Ducalis-io/ducalis-mcp-skills
```

The skill is installed to `.agents/skills/ducalis-use/`. Claude Code reads `SKILL.md` on startup and loads references from disk as needed.

### Other agents

Copy `skills/ducalis-use/` into your agent's skills directory. The agent should read `SKILL.md` first, then load files from `references/` based on the task at hand.

## Prerequisites

You need a running Ducalis MCP server that provides `read_ducalis` and `write_ducalis` tools. See [Ducalis MCP documentation](https://ducalis.io) for setup instructions.

## Updates

```bash
claude skills update
```

## License

MIT
