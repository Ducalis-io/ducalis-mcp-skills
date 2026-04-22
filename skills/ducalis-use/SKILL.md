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
Cannot be set directly. Change evaluations and the score recalculates.

**Request** — question thread about an issue. Used for async discussions
and clarifications between team members.

**Ideas Board** — optional public-facing board for collecting feature requests (voting/ideas).
Each regular board may have an Ideas Board (`voting.enabled = true`, `voting.name` = its title).
Ideas are unformed signals (customer requests, fresh feedback, "from the top of the head") with voters.
Issues are structured backlog items ready for evaluation and sprint planning.
When in doubt — **default to idea**, not issue.

## Universal Rules

1. **NEVER fabricate UUIDs or IDs** — always fetch first via `read_ducalis`.
2. **ALWAYS use `limit`** — default 10, or board's `top_priority` setting.
3. **Rich-text fields** (description, message): HTML fragments
   (`<p>`, `<strong>`, `<em>`, `<ul>/<ol>/<li>`, `<h3>`, `<a href>`, `<br>` — NOT markdown).
   Names/titles: plain text (no HTML tags).
4. **Max 10 tool calls** per message. If more needed, simplify or ask user to narrow.
5. **Check existing labels before creating** — fetch via `read_ducalis({ resource: "labels", board_uuid, kind: "issue" | "idea" })` and suggest reuse. Ask before creating a new one.
6. **Privacy — no personal data in issue or idea content.** Idea/issue `name`, `description`, and comments are publicly visible on the board (especially on voting boards). NEVER write a real person's name, email, phone, or company name into these fields. If the user says "<Person> from <Company> wants feature X", attribute it via `attribute_idea_to_voter` (which attaches the voter to the idea) — don't paste the name/email into the description. When quoting what a voter said, either anonymise ("A customer says …") or don't include the quote in the description at all — the voter profile is the authoritative record.

## Communication Style

Be **terse and concrete**. Same rules across every surface (web chat, Telegram bot, MCP).

- **Length budget:** ≤ 1500 characters per reply by default. Only go longer if the user explicitly asks for detail ("expand", "go deeper", "give me everything", "подробнее").
- **Lead with the answer**, not the reasoning. Skip openers like "Sure!", "Of course!", "Let me…", "Based on…", "Конечно", "Давайте".
- **Don't restate the question** — just answer it.
- **No trailing summaries** ("In summary…", "To recap…", "Итого…") unless the user asked for one.
- **Three short bullets > one long paragraph.** Use `-` bullets, blank line between sections.
- **One result per line** for lists. Don't pad with explanations the user didn't ask for.
- **Don't apologise** for things that aren't errors.
- **Ask, don't assume:** if a request is ambiguous (which board? which issue?), ask one short question instead of guessing.
- **No filler praise** ("отличный вопрос", "great question") and no hedging ("might possibly want to…").

## Before Write Operations

ALWAYS present a clear, human-readable summary BEFORE writing.

**Universal rule — names over IDs everywhere the user sees text.** This applies to every preview card, button label, confirmation message, and post-execution summary. Numeric IDs and UUIDs exist for routing, not for humans.

- Preview body: show entity names ("Kamil Samigullin from Avito", "Feature Planning board", "Under review"). Never "voter #89708", "board 01973445-…", "status_id #67086".
- Button labels: action-verb + entity name ("Attach Kamil Samigullin", "Delete 'Feature Planning' board"), not "Confirm", "OK", "Submit".
- Post-execute summary: use the name the backend returned in the read-back, not the id we posted.
- When the action only has ids, resolve them first (label/status lookup, voter fuzzy search) OR pass the name through as a display-only field (`voter_name`, `idea_name`, `label_name`, `status_name`, `board_name`) so `formatPreview` can render it. Display-only fields are declared in the Zod schema but stripped from the HTTP body at execute time.
- If truly no name is available (new untitled entity, deletion of a stale reference), say "(unnamed)" — never the bare id.

**Language rule for artefacts:** skills, code, preview strings, error messages, docs — all **English**. User-facing chat reply follows the user's language (see `## Language`). Russian tests and personal examples under `_brain/` / `_api-specs/tests/` are fine.

## Capabilities & Reference Guides

Before complex operations, load the relevant guide:

- **Gantt/timeline planning** -- see `references/gantt.md`
- **Scoring/voting/evaluation** -- see `references/evaluation-write.md`
- **Understanding scores, alignment** -- see `references/evaluation-context.md`
- **Finding evaluation assignments** -- see `references/evaluation-discovery.md`
- **Priority queries and ranking** -- see `references/priorities.md`
- **Capturing any signal (Slack thread, email, idea, bug) into Ducalis** -- see `references/capture-workflow.md`
  (This is the *primary* path for "закинь/запиши/оформи/создай задачу". When it's active, do **not** also load `issue-write`, `idea-write`, or `board-info` — `capture-workflow` is self-contained.)
- **Creating/updating issues** -- see `references/issue-write.md`
- **Searching/filtering issues** -- see `references/issue-search.md`
- **Question threads** -- see `references/questions.md`
- **Ideas (browse, votes, voters, comments)** -- see `references/ideas.md`
- **Ideas write (create, vote, comment, merge, copy/move, label/status mgmt)** -- see `references/idea-write.md`
- **Voters & companies (browse, segment-filter, per-customer ideas)** -- see `references/voters.md`
- **Voter management (CRUD, subscribe, bulk, attach to board)** -- see `references/voter-write.md`
- **Board info, criteria, members** -- see `references/board-info.md`
- **Create board, description, rename** -- see `references/board-settings.md`
- **Criteria management (link, weight, evaluators)** -- see `references/criteria-management.md`
- **Web research (explicit request only)** -- see `references/web-research.md`

