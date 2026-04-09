## Evaluation Discovery — what to evaluate, my progress

### Step 1: identify user
Use user id from current context.

### MY evaluation (pass user_id)

**"What do I need to evaluate?"**
`read_ducalis({ resource: "issues", board_uuid: "...", user_id: <id>, include: ["my_voting_progress", "score"], sort_by: "score", sort_order: "desc", limit: 20 })`
Returns ONLY unevaluated active issues for this user. Add `missing_criteria` to show which criteria are left.

**"How many issues left for me?"**
`read_ducalis({ resource: "issues", board_uuid: "...", user_id: <id>, count: true })`

**"Which boards do I evaluate on?"**
`read_ducalis({ resource: "boards", user_id: <id> })`

### TEAM evaluation (no user_id)

**"Overall board progress"** — shown in meta as `Board evaluation progress: N%`

**"Which issues fully evaluated by team?"**
`read_ducalis({ resource: "issues", board_uuid: "...", include: ["voting_percent", "all_voted"], where: { field: "all_voted", op: "eq", value: true } })`

### Gotchas
- `user_id` on issues: only unevaluated active tasks (excludes done, excludes skipped)
- `user_id` on boards: only boards where user has evaluation assignments
- Without `user_id`: all items (team view)
- `missing_criteria`: names of criteria user hasn't voted on yet
- `my_voting_progress` = "3/5" or "skipped" (personal)
- `voting_percent` / `all_voted` = TEAM progress
- When listing issues, format as `[Issue Name](/issue/{id})`
- For "open evaluation": `[Open Evaluation](/board/{uuid}/unvoted)`
