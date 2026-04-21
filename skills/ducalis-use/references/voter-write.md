## Voter management — create, update, subscribe, bulk

### CRITICAL: Button-based confirmation

Same flow as all writes: call with `confirm: false`, wait for `[WRITE_CONFIRMED]`, then `confirm: true`. Never display the signal in your text.

### Actions

#### CRUD
- **create_voting_user** — `email` (req); optional `name`, `company_id`. Creates a workspace-level voter.
- **update_voting_user** — `voting_user_id` (req) + at least one of `email`, `name`, `company_id`.
- **delete_voting_user** — `voting_user_id` (req). PERMANENT — votes/comments unlinked.

#### Bulk
- **bulk_create_voting_users** — `emails: string[]`. Use for CSV-style import where you have many emails and no per-row metadata. For per-row metadata (different names/companies), use `batch` over `create_voting_user`.

#### Board attach
- **add_voter_to_board** — `board_uuid` + `voting_user_id` (req). Attach an existing voter to a specific board so they can be invited / appear in board voter lists.
- **create_board_voting_user** — `board_uuid` + `email` (req); optional `name`. One-shot: create + attach.
- **subscribe_voters_to_boards** — `voting_user_ids: number[]` + `board_uuids: string[]`. Bulk subscribe voters to multiple boards. Sends notifications based on board settings.
- **unsubscribe_voters_from_boards** — same params, opposite effect.
- **remove_voters_from_board** — `board_uuid` + `voting_user_ids: number[]`; optional `remove` (true = PERMANENT removal, false = deactivate), `idea_id` (limit removal to voters of one specific idea).

### "Создай идею от имени voter" workflow

User: «Создай идею про <feature> от <voter name> из <company name>»

1. **Search voter:**
   ```
   read_ducalis({ resource: "voting_users", query: "<voter name>" })
   ```
   - 0 hits → ask user: "Не нашёл voter с таким именем. Создать нового? Email?"
     → on confirm: `create_voting_user({ email, name: "<voter name>" })` — preview, then on `[WRITE_CONFIRMED]` execute.
   - 1 hit → use as author.
   - 2+ hits → list `name + email + company` and ask which one.

2. **(Optional) Pre-check duplicates:**
   ```
   read_ducalis({ resource: "ideas", query: "<рабочее название>\n\n<описание>", query_mode: "dedup", limit: 5 })
   ```
   (Typesense semantic lookup — RU+EN, cross-board. Add `where: { field: "board_uuid", op: "eq", value: board_uuid }` to scope.)
   If a high-vote duplicate exists → ask "Воткнуть голос voter'а в существующую идею или создать новую?"
   - Existing → `vote_idea({ idea_id, voting_user_id: <voter_id> })`
   - New → continue.

3. **Create idea with author:**
   ```
   write_ducalis({
     action: "create_idea",
     params: {
       board_uuid,
       name: "<idea name>",
       description: "<p>...</p>",
       voting_user_id: <voter_id>,
       vote: 1   // initial vote from author — almost always wanted for "от имени"
     },
     confirm: false
   })
   ```
   Single preview shows resolved author + initial vote flag. On `[WRITE_CONFIRMED]` execute.

**Shortcut:** if you only have an email, you can pass `voting_user_email` directly to `create_idea` — the action resolves it at execute time. Throws if the email isn't a known voter (so you must offer `create_voting_user` first).

### Bulk-import voters from a list

If user pastes a list of emails:
```
write_ducalis({
  action: "bulk_create_voting_users",
  params: { emails: ["a@x.com", "b@x.com", "c@x.com"] }
})
```
Then optionally subscribe them to a board:
```
write_ducalis({
  action: "subscribe_voters_to_boards",
  params: { voting_user_ids: [<ids from previous response>], board_uuids: [<board>] }
})
```

For per-row metadata (CSV with name + company per email), use the `batch` action over `create_voting_user`:
```
write_ducalis({
  action: "batch",
  params: {
    action: "create_voting_user",
    items: [
      { email: "<a@example.com>", name: "<Name 1>", company_id: <id1> },
      { email: "<b@example.com>", name: "<Name 2>", company_id: <id2> }
    ]
  }
})
```

### Reuse-first principle

Before `create_voting_user` — ALWAYS run `voting_users` query/where lookup first. Voters are workspace-scoped and duplicates pollute every board's voter list.

### Gotchas

- Subscribing sends emails (depending on board notification settings). For silent attach use `add_voter_to_board` (board-only, no notifications).
- `remove_voters_from_board` with `remove: true` is PERMANENT for that board's voter relationship — votes from those voters get unlinked. Show explicit "PERMANENT" in preview.
- `company_id` on `create_voting_user` is the `voting_companies.id` (not `internal_id`). Look up first via `voting_companies`.
- `voting_user_email` shortcut on `create_idea` only resolves — it does NOT auto-create. Keep voter creation as a separate confirmed step.
