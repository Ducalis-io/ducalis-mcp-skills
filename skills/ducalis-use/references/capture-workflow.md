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

### Step 1 — Board discovery (ALWAYS FIRST, exactly ONE call)

```
read_ducalis({
  resource: "boards",
  include: ["voting", "members_full", "description_template"],
})
```

Each result has: `uuid`, `name`, `emoji`, `voting.{name,emoji,enabled}`,
`members_full[{id,name,email}]`, `description_template`.

**Board matching rules:**
- Match the user's words against `name` and `voting.name`
  (e.g. "роадмапе" → board named "Roadmap", "баги" → "Bugs").
- If one clear match — proceed. If 2+ plausible — ask user to pick.
- Unknown words in the user's message are **board name filters**, not Ducalis entity types.
  **Never** say "нет такого ресурса".

Display boards when asking:
- Plain: `🚀 Help Center Deploy`
- With ideas board: `🚀 Help Center Deploy → ideas: 💡 Help Center Ideas`

---

### Step 2 — Issue vs Idea

| User wording | Create |
|--------------|--------|
| "задача", "баг", "bug", "issue", "в беклог", "в бэклог" | **Issue** on the main board |
| anything else / unclear | **Idea** — unformed signal |

- Ideas go to the board's ideas board (same `board_uuid`; `voting.enabled` may
  be false — still fine, the idea just lives on the main board).
- Issues go to the main board.

---

### Step 3 — Fetch labels + statuses for the chosen board (ONE call)

```
read_ducalis({ resource: "dictionaries", board_uuid: "<chosen_uuid>" })
```

Returns all of: `label`, `status`, `issue_type`, `idea_label`, `idea_status`.

Pick:
- **For ideas:** `idea_status` with `system_status = "new"`, else the first.
- **For issues:** `status` with `system_status = "new"`, else the first.
- **Labels:** scan `label` (issues) or `idea_label` (ideas), pick 1-3 that are
  clearly relevant to the request. If nothing is clearly relevant — omit labels.

---

### Step 4 — Silently assemble params

Work these out without narrating them in chat:

**Assignee (fuzzy match `members_full` from Step 1 — NO extra API call):**
- Omitted or "на себя" → the `User` id from the Current Context block.
- "на Диму" / "@Надя" → the member whose `name` contains the token
  (case-insensitive). If exactly one match — use it silently; mention the
  resolved name in the preview. If 0 matches — fall back to self. If 2+ —
  pick the closest and note alternatives in the preview.

**Description (HTML, two parts, always in this order):**
1. Structured top part shaped by the board's `description_template` — keep it
   short: 2-4 concise paragraphs (`<p>`), bullet lists where natural. If the
   board has no template, write a brief context/intent paragraph.
2. Raw source at the end:
   ```html
   <hr>
   <h3>Исходное сообщение</h3>
   <blockquote><p>…verbatim Slack thread / email / request…</p></blockquote>
   ```

Never Markdown — rich-text fields are HTML fragments.

**Title:** plain text, one line, 40–90 chars, no HTML, no trailing period.

---

### Step 5 — Preview + ONE write_ducalis call

Final visible text before the confirm card:

> Создам *[идею/задачу]* *[title]* в борде *[Board]*, assignee *[Name]*,
> теги *[labels or —]*. Верно?

Then call exactly once:

```
write_ducalis({
  action: "create_idea" | "create_issue",
  params: { board_uuid, name, description, status_id, assignee_id, label_ids? },
  confirm: false,
})
```

The adapter renders confirm/edit/cancel buttons. Do **not** call `write_ducalis`
again in this turn — the actual write happens outside the LLM when the user
clicks ✅.

---

### HARD rules

- **Exactly one** `write_ducalis({ confirm: false })` per response. Never two.
- **Exactly one** board-discovery `read_ducalis` (Step 1). Never split.
- **Max two** `read_ducalis` calls total (Step 1 + Step 3). Nothing else.
- Silent between tool calls — no narration like "Нашёл борд…", "Сейчас
  проверю статусы…". Only emit the Step 5 preview text at the end.
- Never print `[WRITE_CONFIRMED]` or `[WRITE_EDIT]` in visible text.
- Never call `write_ducalis` with `confirm: true` yourself — execution happens
  outside the LLM after the button click.

---

### Edit flow (button ✏️)

When the user clicks ✏️ *Изменить*, the adapter posts the previous preview as
a quote followed by "Что поменять?". The user's next message ("не на Диму, на
Надю") arrives with that quote already in thread context — re-run Step 4
(assemble params) with the correction applied, then Step 5 (new preview).
