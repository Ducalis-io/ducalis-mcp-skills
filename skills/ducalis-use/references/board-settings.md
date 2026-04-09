## Board Settings — create board, description, rename

### Concept

Board description is a living document (wiki) where the user captures goals, metrics, KPIs, decision-making processes, and criteria rationale. It serves as the foundation for:
- Generating evaluation criteria from documented goals
- Providing context when creating or evaluating issues
- Onboarding new team members to the board's purpose

### Read patterns

**"Board description" / "What is this board about?"**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["description"] })`

**"Board name and emoji"**
`read_ducalis({ resource: "board", board_uuid: "..." })` — defaults include name + emoji

### Write patterns

**Create a new board**
`write_ducalis({ action: "create_board", params: { name: "...", emoji: "...", description: "<p>...</p>", platform_id: <from context> } })`
- `name` required, `emoji` and `description` optional
- `platform_id`: take from `## Current Context` -- `Ducalis platform_id` (always pass it)
- Description format: HTML fragments (`<p>`, `<h3>`, `<ul>`, `<strong>`)
- After creation, link to the new board: `[Board Name](/board/{uuid}/summary)`

**Update board description**
`write_ducalis({ action: "update_board", params: { board_uuid: "...", description: "<p>...</p>" } })`
- Navigate user to description page after update: `[description](/board/{uuid}/description)`

**Rename board / change emoji**
`write_ducalis({ action: "update_board", params: { board_uuid: "...", name: "...", emoji: "..." } })`

### Description writing guidelines
- Format: HTML (`<p>`, `<h3>`, `<ul>/<li>`, `<strong>`, `<em>`, `<a href>`)
- Structure: goals/objectives, key metrics, decision framework, success criteria
- Compact by default (3-6 paragraphs). Expand only when user asks for detail
- If name/emoji don't match the described purpose — proactively suggest renaming

### Workflow tips
- After importing tasks: "write a board description based on existing issues" — read issues first, then compose
- After web research: save findings as board description for future reference
- Board description + criteria setup are complementary — description informs which criteria to link
