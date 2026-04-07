## Issue Search — find and filter issues

### Single issue by ID (NO board needed)

`read_ducalis({ resource: "issue", issue_id: 891749 })`
Returns basic fields (name, status, type, description, assignee). No score/criteria (those need board). Use when user asks about a specific issue.

### Batch & filter patterns

**Specific issues by ID list** (ONE call, not one per issue):
`read_ducalis({ resource: "issues", board_uuid, issue_ids: [891749, 416014, 389238], include: ["score", "criteria_scores", "assignee"] })`

**By assignee email (most reliable — same in Ducalis and tracker):**
`read_ducalis({ resource: "issues", board_uuid, where: { field: "assignee_email", op: "eq", value: "vit@ducalis.io" }, include: ["score", "assignee"], sort_by: "score", sort_order: "desc", limit: 10 })`

**By assignee name (may miss tasks — names differ between Ducalis and tracker):**
`read_ducalis({ resource: "issues", board_uuid, where: { field: "assignee", op: "eq", value: "Vit Mee" }, include: ["score", "assignee"], sort_by: "score", sort_order: "desc", limit: 10 })`

**By assignee ID (may be null for tracker-synced tasks):**
`read_ducalis({ resource: "issues", board_uuid, where: { field: "assignee_id", op: "eq", value: 7 }, include: ["score", "assignee"], sort_by: "score", sort_order: "desc", limit: 10 })`

**Compound filter (AND):**
`read_ducalis({ resource: "issues", board_uuid, where: [{ field: "assignee_id", op: "eq", value: 7 }, { field: "status", op: "neq", value: "Done" }], sort_by: "score", sort_order: "desc", limit: 5 })`

**Set membership (in):**
`read_ducalis({ resource: "issues", board_uuid, where: { field: "status_id", op: "in", value: [1, 3] }, limit: 20 })`

**Recent / sprint / labels / by score:**
- Recent: `sort_by: "create_date", sort_order: "desc"`, include `create_date`
- Sprint: include `sprint, completed`
- Labels: include `labels` — meta auto-includes `board_labels` (unique labels on this board with usage count, sorted by count desc)
- By score: `sort_by: "score", sort_order: "desc"`, include `criteria_scores`
- Label discovery (all IDs for write): `read_ducalis({ resource: "dictionaries", where: { field: "type", op: "eq", value: "label" } })`

### Filter fields
- Email-based: `assignee_email` — most reliable, same across Ducalis and tracker. Use `eq` for exact match.
- Name-based: `assignee`, `reporter`, `status`, `type` — use `contains` for substring. ⚠️ Assignee names may differ between Ducalis and tracker.
- ID-based: `assignee_id`, `reporter_id`, `status_id`, `type_id` — use `eq` or `in` for exact match. ⚠️ `assignee_id` may be null for tracker-synced tasks.

### Navigation
- "Open issue #X" → `[Issue Name](/issue/{id})` — NO API call needed
- After listing → format each as `[Issue Name](/issue/{id})`
- "Open backlog" → `[Open Backlog](/board/{uuid}/summary)`

### Rules
- ALWAYS set limit — never fetch unlimited
- Use `issue_ids` when user provides specific IDs — never loop with single `issue` calls
