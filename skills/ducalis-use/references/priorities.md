## Priorities — show top/important items

### Patterns

**"What's most important?" / "Top priorities"**
1. Find board: `read_ducalis({ resource: "boards" })`
2. Get limit: `read_ducalis({ resource: "board", board_uuid: "...", include: ["top_priority"] })`
3. Fetch: `read_ducalis({ resource: "issues", board_uuid: "...", include: ["score", "voting_percent"], sort_by: "score", sort_order: "desc", limit: <top_priority> })`
ALWAYS use board's `top_priority` value as limit — this is the board owner's setting.

**"Lowest priority items"**
Same steps but `sort_order: "asc"` in step 3.

**"Low confidence scores"**
`read_ducalis({ resource: "issues", board_uuid: "...", include: ["score", "voting_percent"], where: { field: "voting_percent", op: "lt", value: 50 }, sort_by: "score", sort_order: "desc" })`

**"Only fully evaluated items"**
Add `where: { field: "all_voted", op: "eq", value: true }` to any priority query.

### Gotchas
- Every issues response includes `Board evaluation progress: N%`. If < 80%: **WARN** that priorities are preliminary and may change significantly as more evaluations come in.
- If `voting_percent` < 50% on an item — warn that its score is unreliable, not enough evaluations.
- If ALL issues have low voting: state "None fully evaluated yet. Complete evaluation before trusting priorities."
- Format issue names as clickable links: `[Issue Name](/issue/{id})`
