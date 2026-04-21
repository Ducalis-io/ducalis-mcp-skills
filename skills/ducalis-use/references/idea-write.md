## Idea Write — create, update, vote, comment, merge, copy/move

### CRITICAL: Button-based confirmation

NEVER call `write_ducalis` with `confirm: true` without first receiving the `[WRITE_CONFIRMED]` system signal. NEVER display `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in your text.

### Flow

1. User requests an idea change (create/update/vote/comment/merge/etc.)
2. For **create_idea / update_idea**: one `read_ducalis({ resource: "idea_context", board_uuid })` — it returns language, inbox status id+name, all board statuses, all board labels, and the idea description template in one shot. For narrower lookups outside the create flow use `labels` / `statuses` with `kind: "idea"`.
3. Call `write_ducalis({ action, params, confirm: false })` in the SAME response — preview card with OK/Edit appears automatically
4. On `[WRITE_CONFIRMED]` — call the same action with `confirm: true`
5. On `[WRITE_EDIT] <instructions>` — adjust params, call `confirm: false` again

**Default column = inbox.** Don't pass `status_id` unless the user explicitly names a column. Backend auto-assigns the board's inbox (`system_status = "under_review"`). `idea_context.inbox_status_name` is just for the preview; don't send it as `status_id`.

**Content language.** `idea_context.language` is authoritative — write title and description fully in that language. Never mix (Russian title + English body is a bug).

### Actions

#### CRUD
- **create_idea** — `board_uuid` + `name` (req); optional `description` (HTML), `status_id` or `status_name`, `allow_voting`, `voting_user_id`, `vote` (initial vote 0/1), `issue_id` (link to existing issue at create time), `custom_fields` (object)
- **update_idea** — `idea_id` (req); any of `name`, `description`, `status_id`/`status_name`, `allow_voting`, `custom_fields`. `board_uuid` auto-resolved if omitted. Status change auto-creates a system comment on the idea.
- **delete_idea** — `idea_id` (req). PERMANENT — votes, comments lost.

#### Voting
- **vote_idea** / **unvote_idea** — `idea_id` (req); optional `voting_user_id` for admin vote-on-behalf
- **custom_vote_idea** — `idea_id` + `email` (req). Records a vote on behalf of the email holder (creates voting user if missing).
- **change_idea_author** — `idea_id` + `email` (req). Changes ownership permanently.

#### Comments
- **create_idea_comment** — `idea_id` + `message` (HTML, req); optional `parent_id` (reply), `assignee_id`, `open`
- **update_idea_comment** — `idea_id` + `comment_id` + `message`
- **delete_idea_comment** — `idea_id` + `comment_id`. Permanent.

#### Labels (assignment on idea)
- **add_idea_label** / **remove_idea_label** — `idea_id` + (`label_id` OR `label_name`). `label_name` auto-resolved against the board's idea-label dictionary.

#### Linkage / merge
- **link_idea_to_issue** — `idea_id` + `issue_internal_id` (req). NB: this is `issue.internal_id`, NOT the DB id. Read it from `issue.internal_id` before calling.
- **unlink_idea_from_issue** — `idea_id`
- **create_backlog_from_idea** — `idea_id` (req); optional `name`, `description` overrides. Creates a fresh backlog issue and links it to the idea.
- **merge_ideas** — `idea_id` (child) + `parent_idea_id`. Child becomes hidden; votes roll up to parent.
- **unmerge_ideas** — `idea_id`

#### Cross-board copy / move
- **copy_idea** — `idea_id` + `target_board_uuid` + (`target_status_id` OR `target_status_name`). Optional `keep_issue` (default false), `keep_comments` (default true).
- **move_idea** — same payload, but removes from source.
- BEFORE preview: read target board's idea statuses (`read_ducalis({ resource: "statuses", board_uuid: "<target_uuid>", kind: "idea" })` — labels/statuses do not auto-remap, so verify the target_status name exists on the target board).

#### Idea label dictionary (board-scoped)
- **create_idea_label** — `board_uuid` + `name`; optional `color` (#hex)
- **update_idea_label** — `board_uuid` + `label_id`; `name` and/or `color`
- **delete_idea_label** — `board_uuid` + `label_id`; optional `new_id` (reassign affected ideas)

#### Idea status dictionary (board-scoped)
- **create_idea_status** — `board_uuid` + `name`; optional `color`
- **update_idea_status** — `board_uuid` + `status_id`; any of `name`, `color`, `system_status`
- **delete_idea_status** — `board_uuid` + `status_id`; optional `new_id`
- **shift_idea_status** — `board_uuid` + `status_id` + `position` (reorder)
- **set_idea_status_as_system** — `board_uuid` + `status_id` + `system_status` (e.g. `under_review`, `in_progress`, `done`) — binds a custom status to a system meaning

### Reuse-first workflow for labels & statuses

Idea labels and statuses are board-scoped. Before creating a new one:
1. `read_ducalis({ resource: "labels", board_uuid, kind: "idea" })` — see what exists (same for `statuses`)
2. Show the user existing options and ask: reuse, create new, or both?
3. Only on confirmation, call `create_idea_label` / `create_idea_status`

Same rule for statuses — don't pollute the board's status set.

### Batch operations

For 2+ identical writes (e.g. add three labels to an idea, or create five labels), use the `batch` action — one tool call, one preview card. See base skill for syntax.

**CSV import pattern** — bulk-create N ideas in one call:
```
write_ducalis({
  action: "batch",
  params: {
    action: "create_idea",
    items: [
      { board_uuid, name: "Row 1", description: "<p>…</p>", status_id },
      { board_uuid, name: "Row 2", description: "<p>…</p>", status_id },
      …
    ]
  }
})
```
The executed response lists each new id (`Created idea #NNN`) and a `/idea/{id}` link per row. Report those links back to the user so they can open the imported ideas.

### Follow-up writes on JUST-CREATED ideas

`/storage/ideas?board=<uuid>` has an indexing lag of several seconds for brand-new ideas. So after `create_idea` / batch create, if you immediately need to `update_idea`, `add_idea_label`, `create_idea_comment`, `delete_idea`, etc. on those ids — **pass `board_uuid` explicitly** in each call. Without it, the resolver does a `/storage/ideas` scan and throws "Idea not found" until the index catches up.

```
// right after batch create:
write_ducalis({ action: "add_idea_label", params: { board_uuid, idea_id: <just-created-id>, label_id } })
```

For subsequent chat turns where the user says "update that idea I just created", rely on the id captured from the create response, not on a re-read.

### Linkage decision tree

- **Idea explains an existing backlog item?** → `link_idea_to_issue` (uses `issue.internal_id`)
- **Idea should become a backlog item?** → `create_backlog_from_idea` (creates new issue, auto-links)
- **Two ideas describe the same need?** → `merge_ideas` (child → parent)

### Author / vote on behalf of customer

Use `voting_user_id` (admin vote) or `custom_vote_idea` (by email) ONLY when the user explicitly says "Иван проголосовал за X" / "vote on behalf of foo@example.com". Never proactive.

### Создать идею от имени voter (recommended workflow)

When the user says "создай идею от <имя> из <компании>" / "idea request from <email>" — full workflow lives in `voter-write.md`. TL;DR:

1. `read_ducalis({ resource: "voting_users", query: "<имя или email>" })` — search.
2. Disambiguate / confirm new voter creation.
3. (Optional) `read_ducalis({ resource: "ideas", query: "<рабочее название>\n\n<описание>", query_mode: "dedup", limit: 5 })` — duplicate check (Typesense semantic, RU+EN). Scope to a board via `where: { field: "board_uuid", op: "eq", value: board_uuid }` if needed.
4. `create_idea` with `voting_user_id` (preferred) OR `voting_user_email` shortcut + `vote: 1`.

The `voting_user_email` shortcut on `create_idea` resolves email → voter id at execute. It throws if the email isn't a known voter — so always offer `create_voting_user` first when search returns 0.

### Gotchas

- `description`, `message` — HTML fragments (`<p>`, `<strong>`, `<ul>/<li>`, `<a>`), NOT Markdown
- Status names: pass `status_name` ONLY if you've verified it exists on the board (resolver will throw otherwise — guide user to existing options)
- Cross-board move is irreversible without re-copying — warn the user explicitly
- `link_idea_to_issue` uses `internal_id` from issue object, NOT the regular `issue.id`
- After write, link affected ideas as `[Idea Name](/idea/{id})` so the user can open them in the iframe
