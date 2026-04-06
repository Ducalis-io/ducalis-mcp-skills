## Evaluation Context — criteria, scales, how to evaluate

### Patterns

**"What are my criteria?" / "How should I evaluate?"**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["my_criteria"] })`
→ Returns per criterion: `{ name, type, description, scale }`
- `type`: "value" (benefit) or "effort" (cost)
- `description`: what this criterion measures and what each scale value means
- `scale`: discrete values to choose from, e.g. "1,2,3,5,8,13"

**"My scores on an issue"**
`read_ducalis({ resource: "issue", board_uuid: "...", issue_id: 123, include: ["my_criteria_scores"] })`

**"All board criteria" (admin overview)**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["criteria_full"] })`
→ `criteria_full` = ALL criteria with coefficients (admin view). `my_criteria` = only criteria assigned to current user (for evaluation).

### Gotchas
- Criterion `description` contains what each scale value means — ALWAYS show to user before they evaluate
- Coefficient is NOT in my_criteria — it's an admin weight, irrelevant for evaluators
- User MUST pick from the discrete `scale` values — no arbitrary numbers
- Link to scores page: `[see scores](/board/{uuid}/unvoted/scores)`
- Link to specific issues: `[Issue Name](/issue/{id})`
