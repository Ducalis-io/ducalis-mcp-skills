## Idea Write — minimal create/update

### Confirmation

Call `write_ducalis({ action, params, confirm: false })` once in the turn. UI renders OK/Edit buttons. On `[WRITE_CONFIRMED]` re-call with `confirm: true`. On `[WRITE_EDIT] <instructions>` adjust and re-preview. Never display the signals in your text.

### Context fetch (once per create/update)

```
read_ducalis({ resource: "idea_context", board_uuid })
```
Returns `language`, `inbox_status_name`, board statuses, board labels, description template.

- **Default column = inbox.** Omit `status_id` unless the user explicitly names one — backend auto-assigns inbox.
- **Language.** Write title + description fully in `idea_context.language`. Never mix.
- **Labels.** Use `label_names` with exact strings from `idea_context.labels`. Never `label_ids` — board-scoped IDs cannot be inferred.

### create_idea

```
write_ducalis({
  action: "create_idea",
  params: {
    board_uuid,
    name,                 // plain text
    description,          // HTML fragment: <p>, <strong>, <ul>/<li>, <a>. Never Markdown.
    label_names?,         // optional, exact from idea_context.labels
    voting_user_id?,      // attach voter (see voter-write.md)
    vote?,                // 1 = initial vote from that voter
  },
  confirm: false,
})
```

For "create idea from <voter>" — full recipe in **voter-write.md**. TL;DR: resolve voter, then pass `voting_user_id + vote: 1` to `create_idea`.

### update_idea

```
write_ducalis({
  action: "update_idea",
  params: { idea_id, board_uuid?, name?, description?, status_name?, allow_voting? },
  confirm: false,
})
```

### After create / batch create

`/storage/ideas?board=<uuid>` has indexing lag. For any follow-up write on a just-created idea (`update_idea`, `add_idea_label`, `create_idea_comment`), pass `board_uuid` explicitly to avoid "Idea not found".

### Output

After success, link affected ideas as `[Idea name](/idea/{id})` so the user can open them in the iframe.

### Other actions (less common)

- `delete_idea`, `add_idea_label`/`remove_idea_label`, `create_idea_comment`/`update_idea_comment`/`delete_idea_comment`, `link_idea_to_issue`/`unlink_idea_from_issue`, `create_backlog_from_idea`, `merge_ideas`/`unmerge_ideas`, `copy_idea`/`move_idea`, idea-label/status dictionary CRUD — all registered but not in the default recipe. Use only on explicit user request.
