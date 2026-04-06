## Evaluation Write — scoring, voting, and questions

### Confirmation flow

1. User requests a write action
2. Gather all required data via `read_ducalis`
3. Present proposed scores/actions as text with reasoning (scale descriptions, rationale)
4. Call `write_ducalis({ action, params, confirm: false })` **in the same response** — a preview card may appear
5. User confirms (button click sends `[WRITE_CONFIRMED]`, OR user types "да"/"давай"/"ок"/"yes") → call `write_ducalis` with SAME action/params + `confirm: true`
6. User wants changes (`[WRITE_EDIT]` or text instructions) → adjust params, call `confirm: false` again

NEVER display `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in your text — these are hidden system signals.

### Actions

**vote** — Set one criterion score for one issue
- Required: `board_uuid`, `issue_id`, `criterion_id`, `vote`
- Vote value MUST be from the criterion's `scale` (e.g. "1,2,3,5,8,13")

**batch_vote** — Set multiple scores at once (preferred for evaluation)
- Required: `board_uuid`, `votes[]` — each with `issue_id`, `criterion_id`, `vote`

**skip_issue** — Skip issue from evaluation
- Required: `board_uuid`, `issue_id`

**unskip_issue** — Resume evaluation for a skipped issue
- Required: `board_uuid`, `issue_id`
- Optional: `user_id` (for specific user), `all` (true = all users)

**create_question** — Ask a question about an issue (creates a Request)
- Required: `board_uuid`, `issue_id`, `assignee_id`, `message` (HTML — rule 7)
- Check open questions first (questions skill loaded automatically) to avoid duplicates

**reply_question** — Reply to an existing question thread
- Required: `request_id`, `message` (HTML — rule 7)

**resolve_question** — Close a question
- Required: `request_id`

**update_question** — Edit question text
- Required: `request_id`, `message` (HTML — rule 7)

**update_question_message** — Edit a reply in a question thread
- Required: `request_id`, `message_id`, `message` (HTML — rule 7)

### Evaluation workflow

When user asks to evaluate tasks:

**Step 1: Gather data**
1. Read board criteria: `read_ducalis({ resource: "board", board_uuid, include: ["my_criteria"] })`
2. Read unvoted tasks in DEFAULT order (no `sort_by`!): `read_ducalis({ resource: "issues", board_uuid, user_id, include: ["my_voting_progress", "missing_criteria", "description"], limit: 10 })`

**Step 2: Propose scores with scale descriptions**
For each task, show the SCALE DESCRIPTION from the criterion's `description` field, not just a bare number:
- GOOD: "**Retention** = 3 (Moderate — improves retention for some users)"
- BAD: "**Retention** = 3"

Also provide a brief rationale for each score.

**Step 3: Confirm and execute**
After user confirms, call `batch_vote` directly. Then optionally add rationales via `batch` with `create_question` using assignee data already in context.

### Gotchas
- ALWAYS show scale value descriptions when proposing scores
- Do NOT sort tasks when evaluating — default API order matches the Evaluation screen
- Vote values must match criterion `scale` values exactly
- Do NOT vote on criteria not assigned to user (check `my_criteria`)
- Do NOT re-read data already in context — use assignee/reporter from the initial fetch
- `message` в вопросах/ответах — HTML-фрагмент (e.g. `<p>text</p>`), не plain text и не Markdown
- After scoring, link to: `[see scores](/board/{uuid}/unvoted/scores)`
