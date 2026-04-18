## Ideas — browse, filter, inspect feature ideas (a.k.a. VotingIssue)

Ideas are user-submitted feature requests on a board. Each has a status, label set, vote count, optional linked backlog issue, comments, and voting users/companies. Merged children (`parent_id != null`) are hidden by default.

### Default rules

- ALWAYS scope to a board: pass `board_uuid` (or `where: { field: "board_uuid", op: "eq", value: "<uuid>" }`).
- Merged-child ideas are excluded by the resource fetch — no extra filter needed.
- `defaults`: `id`, `name`, `status`, `vote_count`, `author`. Use `include` for richer fields.
- `defaultLimit`: 30. Use `limit` to widen.

### Read patterns

**"Ideas on this board"**
`read_ducalis({ resource: "ideas", board_uuid: "<uuid>", limit: 50 })`

**"Top voted ideas"**
`read_ducalis({ resource: "ideas", board_uuid: "<uuid>", sort_by: "vote_count", sort_order: "desc", limit: 20 })`

**Filter by status (name OR id):**
`read_ducalis({ resource: "ideas", board_uuid: "<uuid>", where: [{ field: "status", op: "eq", value: "Under review" }] })`
- Use `status_id` (in `where`) when you already have the id from `dictionaries`/`statuses` meta.

**Filter by label:**
`read_ducalis({ resource: "ideas", board_uuid: "<uuid>", include: ["labels"], where: { field: "labels", op: "contains", value: "ux" } })`

**Ideas without linked issue:**
`read_ducalis({ resource: "ideas", board_uuid: "<uuid>", where: { field: "linked_issue_id", op: "eq", value: null } })`
- `eq null` matches fields that are null/absent.

**Complex AND filter (e.g. "bug ideas in Inbox without a linked issue"):**
```
where: [
  { field: "status", op: "eq", value: "Inbox" },
  { field: "labels", op: "contains", value: "bug" },
  { field: "linked_issue_id", op: "eq", value: null }
]
```
- `contains` on an array field (`labels`) checks membership; case-insensitive on strings.
- To only count: pass `count: true` in the same call.

**My ideas (authored by current user):**
Pass `user_id: <current_user_id>` — ideas authored by them OR voted by them are returned (authored come first).

**Detail view (single idea):**
`read_ducalis({ resource: "idea", issue_id: <idea_id> })`
- Detail loads voters, companies, comments, linked issue automatically.
- Use `include: ["voting_users", "companies", "comments"]` to opt in selectively when calling with `board_uuid`.

### Comments

For a flat list of all idea comments on a board:
`read_ducalis({ resource: "idea_comments", board_uuid: "<uuid>", limit: 50 })`
- `defaults`: `id`, `voting_issue_id`, `idea_name`, `author`, `message`.
- Filter to one idea: `where: { field: "voting_issue_id", op: "eq", value: <idea_id> }`.

### Status / label dictionary

Idea-side dictionaries are exposed alongside issue ones:
`read_ducalis({ resource: "dictionaries", where: { field: "type", op: "eq", value: "idea_status" } })`
`read_ducalis({ resource: "dictionaries", where: { field: "type", op: "eq", value: "idea_label" } })`

### Navigation

Each idea has `link: "/idea/{id}"`. Render as `[Idea Name](/idea/{id})` — chat adapter intercepts and opens the idea inside the iframe.

### Common gotchas

- `voting_user_id` is the AUTHOR (a `VotingUser`, not a workspace user). Workspace `user_id` and `voting_user_id` are different namespaces.
- `vote_count` is aggregate; individual votes for the current user come from `my_voted` (boolean).
- Status NAMES are board-scoped — same name can have different ids on different boards. Always resolve per board.
- `parent_id != null` means the idea is merged into another — these are filtered out by default.
