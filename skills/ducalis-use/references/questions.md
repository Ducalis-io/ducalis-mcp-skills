## Questions — browse and manage evaluation questions

Questions are tied to issues. When someone can't understand an issue during evaluation, they ask a question — the issue leaves their evaluation queue until resolved.

### Default: open questions only

ALWAYS add `where: { field: "open", op: "eq", value: true }` unless user explicitly asks for closed/resolved/all questions.

### Board-scoped questions

When user asks about "questions on this board" — add board_uuid filter:
`read_ducalis({ resource: "requests", where: [{ field: "open", op: "eq", value: true }, { field: "board_uuid", op: "eq", value: "<uuid>" }], limit: 50 })`

Use `limit: 50` to see all questions (boards usually have <50 open). Message text (up to 200 chars) is always shown — enough to compare and detect duplicates.

### Category patterns (match UI tabs)

Use current user id from context. `assignee_id` and `author_id` are always in the response — no need to include them.

**"Questions to me" / "вопросы ко мне":**
`where: [{ field: "open", ..., value: true }, { field: "assignee_id", op: "eq", value: USER_ID }]`

**"My questions" / "мои вопросы":**
`where: [{ field: "open", ..., value: true }, { field: "author_id", op: "eq", value: USER_ID }]`

**"Other" / "другие вопросы":**
Both `assignee_id neq USER_ID` AND `author_id neq USER_ID`

### Duplicate detection

Duplicates = same `issue_id` + identical or near-identical `message`. To find duplicates:
1. Fetch ALL open questions for the board (limit: 50)
2. Group by `issue_id` — issues with 2+ questions are candidates
3. Compare message text within each group
4. Close duplicates via `resolve_question`, keep one per issue

### View question thread

`read_ducalis({ resource: "request", issue_id: REQUEST_ID, include: ["messages", "issue_description", "issue_status"] })`

### Edit question text

Use `update_question` to rewrite a question directly — no need to close and recreate:
`write_ducalis({ action: "update_question", params: { request_id: ID, message: "<p>new text</p>" }, confirm: false })`

Use `update_question_message` to edit a reply:
`write_ducalis({ action: "update_question_message", params: { request_id: ID, message_id: MSG_ID, message: "<p>new</p>" }, confirm: false })`

### Rules

- Default to open questions — closed are hidden in UI
- ALWAYS set `limit` (default: 20, use 50 for full board scan)
- `assignee_id` and `author_id` are in defaults — use them directly
- `issue_name` always shown — no extra include needed
- For creating/replying/resolving → see evaluation-write skill
