## Issue Write — create, update, delete issues and manage properties

### CRITICAL: Button-based confirmation

NEVER call `write_ducalis` with `confirm: true` without first receiving the `[WRITE_CONFIRMED]` system signal. NEVER display `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in your text — they are hidden UI signals.

### Flow

1. User requests an issue change (create, edit, delete, label, etc.)
2. For **create_issue / update_issue**: one `read_ducalis({ resource: "issue_context", board_uuid })` — returns board language, default_status id+name, all statuses, all labels, all issue types, and the issue description template in one shot. For other actions use narrower reads (`labels` / `statuses` with `kind: "issue"`, or `issues` / `issue`).
3. Call `write_ducalis({ action, params, confirm: false })` **in the same response** — preview card with OK/Edit buttons appears automatically. Do NOT ask for text confirmation.
4. User clicks OK: you receive `[WRITE_CONFIRMED]`. Immediately call `write_ducalis` with the SAME action and params + `confirm: true`
5. User clicks Edit: you receive `[WRITE_EDIT] <instructions>`. Adjust params per instructions, call `confirm: false` again

**Default status.** Don't pass `status_id` unless the user explicitly names a column. Backend auto-assigns the board's default status (`is_default = true`, `system_status = "todo"`).

**Content language.** `issue_context.language` is authoritative for title + description. Never mix languages.

### Actions

**create_issue** — Create a new issue on a board
- Required: `board_uuid`, `name`
- Optional: `description` (HTML — rule 7), `assignee_id`, `reporter_id`, `status_id`, `type_id`, `custom_fields`
- **Before preview** — run a Typesense de-dup lookup so duplicates surface to the user before the create card:
  `read_ducalis({ resource: "issues", query: "<name>\n\n<plain description>", query_mode: "dedup", limit: 5 })`
  If close matches exist → ask the user "Похоже на уже существующее — обновим / привяжем / всё-таки создадим новое?" before calling `create_issue`.

**update_issue** — Update issue fields (only include fields being changed)
- Required: `board_uuid`, `issue_id`
- Optional: `name`, `description` (HTML — rule 7), `assignee_id`, `reporter_id`, `status_id`, `type_id`, `custom_fields`

**delete_issue** — Delete an issue (PERMANENT — always warn user)
- Required: `board_uuid`, `issue_id`

**add_label** / **remove_label** — Manage issue labels (SEPARATE endpoints, NOT part of update_issue)
- Required: `board_uuid`, `issue_id`, + either `label_id` or `label_name` (auto-resolved)

**create_label** — Create a new workspace-level label
- Required: `name`
- Optional: `color` (named token: red, blue, green, orange, amber, yellow, grass, teal, cyan, sky, indigo, violet, plum, crimson, brown, tomato)

**add_watcher** / **remove_watcher** — Manage issue watchers
- Required: `board_uuid`, `issue_id`, `user_id`

**link_issues** / **unlink_issues** — Manage issue dependencies
- Required: `board_uuid`, `issue_id`, `linked_issue_id`
- `link_issues` also requires `type`: `link` (related) or `block` (blocks/depends on)
- `unlink_issues`: `type` is optional

### Label workflow — ALWAYS batch, ALWAYS discover first

Labels are workspace-scoped (shared across ALL boards). Creating duplicates pollutes the dictionary for every user.

**Mandatory flow (4-5 tool calls max for any label operation):**

1. **Discover** — read board issues + board-scoped labels in parallel:
   - `read_ducalis({ resource: "issues", board_uuid, include: ["labels"], limit: 1 })` — meta has `board_labels: [{name, count}]`
   - `read_ducalis({ resource: "labels", board_uuid, kind: "issue" })` — full label set with IDs
2. **Analyze & present** — tell the user what you found:
   - Which labels exist on this board (from board_labels meta)
   - Which workspace labels could be reused
   - Which new labels would need to be created
   - Ask: *"Reuse existing, create new, or both?"*
3. **Batch create** missing labels (ONE tool call):
   `write_ducalis({ action: "batch", params: { action: "create_label", items: [{name: "resource", color: "blue"}, {name: "skill", color: "green"}, ...] } })`
4. **Batch assign** labels to issues (ONE tool call):
   `write_ducalis({ action: "batch", params: { action: "add_label", items: [{board_uuid, issue_id: 123, label_name: "resource"}, {board_uuid, issue_id: 123, label_name: "skill"}, ...] } })`

**CRITICAL — use batch, not individual calls:**
- ✗ NEVER call `create_label` 12 times — use ONE `batch` with `create_label` items
- ✗ NEVER call `add_label` 20 times — use ONE `batch` with `add_label` items
- ✗ NEVER create labels without first checking dictionary
- ✗ NEVER create a label when a similar one exists ("UI" exists -- don't create "ui")
- ✗ NEVER pass labels to `update_issue` — it silently ignores them

### Batch operations

For 2+ identical operations, use the `batch` action (see base skill for syntax). Present all items as text for user confirmation first. One tool call, one UI card.

### Prerequisite reads — resolve IDs before writing

- **Board UUID**: taken from the page context when available; otherwise `read_ducalis({ resource: "boards", include: ["voting"] })`
- **Assignee/reporter**: `read_ducalis({ resource: "board", board_uuid, include: ["members"] })`
- **Labels / statuses / types**: `read_ducalis({ resource: "issue_context", board_uuid })` for create/update (all three + template in one hit), or narrow `labels` / `statuses` with `board_uuid + kind: "issue"`
- **Issue existence**: `read_ducalis({ resource: "issue", board_uuid, issue_id })` — verify before update/delete

### Gotchas

- Labels and watchers use SEPARATE add/remove endpoints — do NOT pass them in `update_issue`
- NEVER delete without explicit warning about permanence + user confirmation
- When user says "assign to X", resolve X to `user_id` via board members first
- Only include changed fields in `update_issue` — omit unchanged fields
- `description` — HTML-фрагмент (e.g. `<p>text</p><ul><li>item</li></ul>`), не Markdown
- After write, include markdown links: `[#ID](/issue/{id})`
