## Capture Workflow — turn any signal into an issue or idea

This is the **primary** path for capturing Slack threads, emails or customer
messages into Ducalis. When this module is active, do NOT also consult
`issue-write.md`, `idea-write.md`, or `board-info.md` — everything you need is
below.

### When to use

Any time the user writes "закинь", "запиши", "скинь", "оформи", "зафиксируй",
"добавь", "создай задачу/идею", "capture", or shares a Slack thread / email /
customer message to save into Ducalis.

---

### Step 1 — Resolve the board (ZERO fetch when page context has one)

Three branches, pick exactly one:

1. **Current Context shows a voting board** (`pageType = "voting_board"` + `boardUuid`) — use that `boardUuid` directly. This is the idea flow. **No read_ducalis call.** Skip to Step 2.
2. **Current Context shows a regular board / issues page** (`boardUuid` present, not a voting board) — use that uuid. This is the issue flow. **No read_ducalis call.** Skip to Step 2.
3. **No board in context** (dashboard, unknown page) — and only then:
   ```
   read_ducalis({ resource: "boards", include: ["voting"] })
   ```
   Match the user's words against `name` / `voting.name`. If unique — pick it.
   If 2+ plausible — ask the user to choose. Never add other `include` fields
   here — labels, statuses, templates come from Step 3, not this list.

**Never** say "нет такого ресурса" — unknown words in the user's message are
board-name filters, not Ducalis entity types.

---

### Step 2 — Issue vs Idea

| User wording | Create |
|--------------|--------|
| "задача", "баг", "bug", "issue", "в беклог", "в бэклог" | **Issue** flow |
| anything else / unclear | **Idea** flow (default on voting boards) |

On a voting board the default is **idea** (that's the point of the public
roadmap). On a regular board the default is **issue**.

---

### Step 3 — Fetch full board context in ONE call

Depending on the branch:

- **Idea flow:**
  ```
  read_ducalis({ resource: "idea_context", board_uuid: "<uuid>" })
  ```
  Returns: `board_name`, `board_emoji`, `language`, `description_template`
  (if enabled), `inbox_status_id` + `inbox_status_name`, `statuses[]` (each
  with `is_inbox`), `labels[]`.

- **Issue flow:**
  ```
  read_ducalis({ resource: "issue_context", board_uuid: "<uuid>" })
  ```
  Returns: `board_name`, `board_emoji`, `language`, `description_template`
  (issue template), `default_status_id` + `default_status_name`, `statuses[]`
  (each with `is_default`), `labels[]`, `issue_types[]`.

**Never** call `resource: "dictionaries"` — that resource no longer exists.
For a standalone label/status lookup outside the create flow, use
`labels` / `statuses` with `board_uuid` + `kind: "idea" | "issue"`.

---

### Step 4 — Silently assemble params

**Content language.** Write the title **and** description fully in
`context.language` (fallback `"en"` when null). Never mix languages —
Russian title + English body is a bug. If the idea template placeholders are
in language X, write answers in language X. The language of the chat with the
admin stays Russian/whatever it is — that's only for the preview text, not
for the content that goes into Ducalis.

**Status — idea flow:**
- If the user **explicitly** names a column ("в Met prioriteit", "в Backlog") —
  find the matching status in `context.statuses`, send `status_id`.
- Otherwise — **do not pass `status_id` at all**. Backend auto-assigns the
  inbox column (`is_inbox: true`). This is the correct default.

**Status — issue flow:**
- Same principle. Explicit column → resolve via `context.statuses`,
  send `status_id`. Otherwise omit — backend defaults to
  `default_status_id`.

**Labels:** from `context.labels`, pick 1–3 semantically relevant entries.
Prefer specific labels (e.g. `AI`, `MCP`) over general ones (e.g.
`Integrations`) when both fit. If nothing is clearly relevant — omit.

**Description (HTML, two parts, always in this order):**
1. Structured top part following `context.description_template` when present.
   Keep it short: 2–4 concise paragraphs (`<p>`), bullet lists where natural.
   If there's no template — brief context/intent paragraph.
2. Raw source at the end:
   ```html
   <hr>
   <h3>Original message</h3>
   <blockquote><p>…verbatim Slack thread / email / request…</p></blockquote>
   ```
   Heading and quote stay in the board's language too.

Never Markdown — rich-text fields are HTML fragments.

**Title:** plain text, one line, 40–90 chars, no HTML, no trailing period,
in the board's language.

**Assignee (issue flow only — fuzzy match from page-context members if
already shown; otherwise skip or use Current Context's user id):**
- Omitted or "на себя" → the `User` id from the Current Context block.
- Explicit name — don't fetch members for this; the user can correct via
  Edit button if needed.

---

### Step 5 — Preview + ONE write_ducalis call

The preview text addresses the admin in **their** language (the chat
language), but `title` and a snippet of `description` appear verbatim in the
board's language. That visible language switch is correct — it shows the
admin what actually lands in Ducalis.

For the **idea** flow:
> Создам идею в *[board_name]* (колонка: *[inbox_status_name]*, язык:
> *[language]*): "*[title]*" — labels: *[…]*. Подтверждаешь?

For the **issue** flow:
> Создам задачу в *[board_name]* (статус: *[default_status_name]*, язык:
> *[language]*): "*[title]*" — labels: *[…]*, assignee: *[name]*. Подтверждаешь?

Then call exactly once:

```
write_ducalis({
  action: "create_idea" | "create_issue",
  params: { board_uuid, name, description, label_ids?, status_id?, /* + assignee_id for issue */ },
  confirm: false,
})
```

The adapter renders confirm/edit/cancel buttons. Do **not** call
`write_ducalis` again in this turn — the actual write happens outside the LLM
when the user clicks ✅.

---

### HARD rules

- **Exactly one** `write_ducalis({ confirm: false })` per response.
- **Zero or one** board-discovery `read_ducalis` (Step 1, only if page
  context has no `boardUuid`).
- **Exactly one** context `read_ducalis` (Step 3 — `idea_context` or
  `issue_context`).
- **Max two** `read_ducalis` calls total. Never split labels/statuses
  into separate reads.
- Never call `resource: "dictionaries"` — it no longer exists.
- Never pass `include: ["description_template", "idea_description_template"]`
  to `boards` — templates live on context resources now.
- Never pass `status_id` unless the user explicitly named the column.
- Silent between tool calls — no narration like "Нашёл борд…", "Сейчас
  проверю статусы…". Only emit the Step 5 preview text at the end.
- Never print `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in visible text.
- Never call `write_ducalis` with `confirm: true` yourself — execution
  happens outside the LLM after the button click.

---

### Edit flow (button ✏️)

When the user clicks ✏️ *Изменить*, the adapter posts the previous preview as
a quote followed by "Что поменять?". The user's next message ("не на Диму, на
Надю") arrives with that quote already in thread context — re-run Step 4
(assemble params) with the correction applied, then Step 5 (new preview).
The context resource result from the earlier turn is usually still fresh —
you don't need to re-fetch `idea_context` / `issue_context` unless the
board changed.
