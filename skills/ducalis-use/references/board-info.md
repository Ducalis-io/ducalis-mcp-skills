## Board Info — board details, settings, formula

### Patterns

**"Show my boards" / "Find a board"**
`read_ducalis({ resource: "boards" })`

**"Board details"**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["description", "criteria_full", "scoring", "sprint"] })`

**"Board formula / scoring method"**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["scoring"] })`

**"Board members / How many?"**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["member_count", "members"] })`

**"Count boards"**
`read_ducalis({ resource: "boards", count: true })`

**"Board with most members"**
`read_ducalis({ resource: "boards", sort_by: "member_count", limit: 1 })`

### Gotchas
- Board names may contain emoji — match by name text, not emoji
- When listing boards, format each as a clickable markdown link: `[Board Name](/board/{uuid}/summary)`
