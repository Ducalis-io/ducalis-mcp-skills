## Voter Write — minimal "create idea from voter"

### Confirmation

Same flow as all writes: `confirm: false` → UI buttons → `[WRITE_CONFIRMED]` → `confirm: true`. Never display signals.

### CRITICAL: pass display-only names

Every voter-related write accepts optional `voter_name`, `voter_email`, `board_name`, `idea_name`. Pass them from search results so the preview reads "Create idea 'X' for Francesco <fmm@consensus.dk>" instead of "voter #89708". Stripped before HTTP.

### Primary recipe — "Создай идею от <voter>"

User: «Созвонился с <name> из <company>, они хотят <feature>. Создай идею на <board>.»

1. **Search voter:**
   ```
   read_ducalis({ resource: "voting_users", query: "<name or email>" })
   ```
   - 0 hits → ask: «Не нашёл voter. Создать нового? Какой email?» → `create_voting_user({ email, name })` with preview.
   - 1 hit → use as author.
   - 2+ hits → list `name + email + company`, ask which.

2. **Create idea with voter attached (ONE call, ONE preview):**
   ```
   write_ducalis({
     action: "create_idea",
     params: {
       board_uuid,
       name,
       description,           // HTML, in idea_context.language
       voting_user_id: <id>,  // NOT voter_id — strict schema uses voting_user_id
       vote: 1,               // initial vote from that voter
       voter_name,            // display-only
       voter_email,           // display-only
       board_name,            // display-only
     },
     confirm: false,
   })
   ```
   Single preview card shows resolved author + vote flag. On `[WRITE_CONFIRMED]` execute.

**Shortcut — only email:** pass `voting_user_email` instead of `voting_user_id`. Resolves at execute; throws if unknown → offer `create_voting_user` first.

### Voter CRUD

- `create_voting_user` — `email` (req); optional `name`, `company_id`.
- `update_voting_user` — `voting_user_id` (req) + at least one of `email`, `name`, `company_id`.
- `delete_voting_user` — `voting_user_id`. PERMANENT.

### Attach voter to an existing idea

Only when the idea already exists:
```
write_ducalis({
  action: "attribute_idea_to_voter",
  params: { idea_id, voter_id, vote: 1, voter_name, voter_email, idea_name },
  confirm: false,
})
```

### Reuse-first

Always `read_ducalis({ resource: "voting_users", query: ... })` before `create_voting_user`. Voters are workspace-scoped; duplicates pollute every board.

### Other actions (less common)

`bulk_create_voting_users`, `add_voter_to_board`, `create_board_voting_user`, `subscribe_voters_to_boards`, `unsubscribe_voters_from_boards`, `remove_voters_from_board` — registered but not in the default recipe. Use only on explicit user request.
