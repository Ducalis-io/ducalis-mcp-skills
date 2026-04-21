## Capture Workflow — turn any signal into an issue or idea

### When to use

Any time the user wants to "закинь", "запиши", "скинь", "оформи", "добавь", "capture", or shares
a Slack thread / email / customer message to put into Ducalis.

---

### Step 1: Board discovery (ALWAYS run first — never skip)

```
read_ducalis({ resource: "boards", include: ["voting"] })
```

Each result has: `uuid`, `name`, `emoji`, and `voting` (with `name`, `enabled`).

Display format:
- Board without ideas: `🚀 Help Center Deploy`
- Board with ideas board: `🚀 Help Center Deploy → ideas: 💡 Help Center Ideas`

**Matching rules:**
- If the user's text implies a topic ("роадмапе", "релиз", "bugs", "фичи") — find the board(s) whose `name` or `voting.name` contains that word. Pick the best match and state which one you chose.
- If multiple boards are equally plausible — show the list and ask user to pick.
- **NEVER say "нет такого ресурса"** — unknown words in the user's message are board name filters, not Ducalis entity types.
- If user describes a topic (e.g. "в роадмапе") in a follow-up message — re-run board discovery and filter by that word.

If no board is found at all — suggest the user create a new board or provide the board name.

---

### Step 2: Issue vs Idea — default to idea

| User says | Create |
|-----------|--------|
| "задачу", "баг", "bug", "issue", "в беклог", "в бэклог" | **Issue** on the main board |
| anything else / unclear | **Idea** — treat as an unformed signal |

**Ideas** = unformed feedback, "из головы", customer signals, feature requests. They have voters, belong to a public board.
**Issues** = structured backlog items, ready for prioritization/evaluation, have assignees and criteria.

For ideas: use the board's ideas board (`voting.name`, same `board_uuid`).
For issues: use the main board.

---

### Step 3: Confirm with user (confirmation buttons)

Before writing, show:
> "Создам **[идею/задачу]** *[suggested title]* в борде *[Board Name]*, назначу на тебя. Верно?"

Use `write_ducalis({ action: "create_idea" | "create_issue", params: {...}, confirm: false })`.
Wait for `[WRITE_CONFIRMED]` before calling with `confirm: true`.

If the user wants a different board or assignee — adjust and show the preview again.

---

### Step 4: Fetch templates and statuses before writing

**For Ideas:**
```
read_ducalis({ resource: "dictionaries", board_uuid, where: { field: "type", op: "eq", value: "idea_status" } })
```
Find the "incoming" status: look for `system_status = "new"` or the first status in the list.
Use that `status_id` when creating the idea (ideas land in the first/incoming column by default).

**For Issues:**
```
read_ducalis({ resource: "board", board_uuid, include: ["description_template"] })
```
If `description_template` is non-empty — use it as the structure for the formatted description.

---

### Step 5: Two-pass write

**Pass 1 — Raw**: Create with `description` = verbatim source text (the Slack thread, email, user's words as-is).
This preserves the original in revision history.

**Pass 2 — Formatted**: Immediately call `update_idea` / `update_issue` with `description` reformatted
per the board's template (and in the template's language — ru/en as the template uses).

Both passes go through the normal confirm flow.
If no template exists — skip Pass 2; the raw text IS the description.

---

### Gotchas

- `voting.enabled = false` → board has no public ideas board; create ideas there anyway (they just won't be public)
- After Pass 1 create, use the returned `idea_id` / `issue_id` directly in Pass 2 (don't re-search — indexing lag)
- Always link result: `[Idea Name](/idea/{id})` or `[#ID](/issue/{id})`
