## Issue Write — create, update, delete issues and manage properties

### CRITICAL: Button-based confirmation

NEVER call `write_ducalis` with `confirm: true` without first receiving the `[WRITE_CONFIRMED]` system signal. NEVER display `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in your text — they are hidden UI signals.

### Flow

1. User requests an issue change (create, edit, delete, label, etc.)
2. Gather required data via `read_ducalis` — resolve IDs from names
3. Call `write_ducalis({ action, params, confirm: false })` **in the same response** — preview card with OK/Edit buttons appears automatically. Do NOT ask for text confirmation.
4. User clicks OK → you receive `[WRITE_CONFIRMED]` → immediately call `write_ducalis` with the SAME action and params + `confirm: true`
5. User clicks Edit → you receive `[WRITE_EDIT] <instructions>` → adjust params per instructions, call `confirm: false` again

### Actions

**create_issue** — Create a new issue on a board
- Required: `board_uuid`, `name`
- Optional: `description` (HTML — rule 7), `assignee_id`, `reporter_id`, `status_id`, `type_id`, `custom_fields`

**update_issue** — Update issue fields (only include fields being changed)
- Required: `board_uuid`, `issue_id`
- Optional: `name`, `description` (HTML — rule 7), `assignee_id`, `reporter_id`, `status_id`, `type_id`, `custom_fields`

**delete_issue** — Delete an issue (PERMANENT — always warn user)
- Required: `board_uuid`, `issue_id`

**add_label** / **remove_label** — Manage issue labels (SEPARATE endpoints, NOT part of update_issue)
- Required: `board_uuid`, `issue_id`, `label_id`

**add_watcher** / **remove_watcher** — Manage issue watchers
- Required: `board_uuid`, `issue_id`, `user_id`

**link_issues** / **unlink_issues** — Manage issue dependencies
- Required: `board_uuid`, `issue_id`, `linked_issue_id`
- `link_issues` also requires `type`: `link` (related) or `block` (blocks/depends on)
- `unlink_issues`: `type` is optional

### Batch operations

For 2+ identical operations, use the `batch` action (see base skill for syntax). Present all items as text for user confirmation first. One tool call → one UI card.

### Prerequisite reads — resolve IDs before writing

- **Board UUID**: `read_ducalis({ resource: "boards" })` — always step 1
- **Assignee/reporter**: `read_ducalis({ resource: "board", board_uuid, include: ["members"] })`
- **Labels/status/type IDs**: `read_ducalis({ resource: "issues", board_uuid, include: ["labels"], limit: 1 })` — dictionaries in context
- **Issue existence**: `read_ducalis({ resource: "issue", board_uuid, issue_id })` — verify before update/delete

### Gotchas

- Labels and watchers use SEPARATE add/remove endpoints — do NOT pass them in `update_issue`
- NEVER delete without explicit warning about permanence + user confirmation
- When user says "assign to X", resolve X to `user_id` via board members first
- Only include changed fields in `update_issue` — omit unchanged fields
- `description` — HTML-фрагмент (e.g. `<p>text</p><ul><li>item</li></ul>`), не Markdown
- After write, include markdown links: `[#ID](/issue/{id})`
