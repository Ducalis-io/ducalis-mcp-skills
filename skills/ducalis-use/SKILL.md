---
name: ducalis-use
description: >
  Work with Ducalis — a B2B SaaS tool for backlog prioritization.
  Read/write boards, issues, evaluations, Gantt timelines, questions.
  Load this skill before working with Ducalis data.
---

# Ducalis — Backlog Prioritization Tool

Use `read_ducalis`, `write_ducalis`, and `web_research` tools to interact with Ducalis.

## Language

ALWAYS respond in the SAME language the user writes in.

## Boundaries

Use ONLY the provided tools. Never fabricate data — always fetch first.
If a capability is not listed, say what you CAN do.

## Core Concepts

**Board** — prioritization workspace (UUID-based). Contains issues, evaluation
criteria, and team members.

**Issue** — backlog item. Has title, description (HTML), status, assignee,
labels, and computed priority score.

**Criterion** — evaluation dimension (e.g. "Revenue Impact", "Technical Risk").
Each has: type (value_criterion or effort_criterion), discrete scale,
and assigned_users (= evaluators, NOT the issue assignee).

**Priority Score** — COMPUTED from evaluations via framework formula.
Cannot be set directly. Change evaluations → score recalculates.

**Request** — question thread about an issue. Used for async discussions
and clarifications between team members.

## Universal Rules

1. **NEVER fabricate UUIDs or IDs** — always fetch first via `read_ducalis`.
2. **ALWAYS use `limit`** — default 10, or board's `top_priority` setting.
3. **Rich-text fields** (description, message): HTML fragments
   (`<p>`, `<strong>`, `<em>`, `<ul>/<ol>/<li>`, `<h3>`, `<a href>`, `<br>` — NOT markdown).
   Names/titles: plain text (no HTML tags).
4. **Compact by default** — 2-4 paragraphs unless user asks for detail.
5. **Max 10 tool calls** per message. If more needed, simplify or ask user to narrow.
6. **Labels are workspace-scoped** — before creating new labels, ALWAYS fetch existing ones first (`resource: "dictionaries"`) and suggest reuse. Ask before creating.

## Before Write Operations

ALWAYS present a clear, human-readable summary BEFORE writing:
- Use tables or structured lists, not raw IDs/UUIDs
- Show entity NAMES: "T2-1 Research → T2-2 Resource" (not issue_id: 3623356)
- Explain what will happen: "Create 11 dependencies linking these task chains"

## Capabilities & Reference Guides

Before complex operations, load the relevant guide:

- **Gantt/timeline planning** → load `references/gantt.md`
- **Scoring/voting/evaluation** → load `references/evaluation-write.md`
- **Understanding scores, alignment** → load `references/evaluation-context.md`
- **Finding evaluation assignments** → load `references/evaluation-discovery.md`
- **Priority queries and ranking** → load `references/priorities.md`
- **Creating/updating issues** → load `references/issue-write.md`
- **Searching/filtering issues** → load `references/issue-search.md`
- **Question threads** → load `references/questions.md`
- **Board info, criteria, members** → load `references/board-info.md`
- **Create board, description, rename** → load `references/board-settings.md`
- **Criteria management (link, weight, evaluators)** → load `references/criteria-management.md`
- **Web research (explicit request only)** → load `references/web-research.md`

