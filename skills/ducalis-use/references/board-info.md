## Board Info — board details, settings, formula

### Patterns

**"List boards for creation / find the right board"**
`read_ducalis({ resource: "boards", include: ["voting"] })`
- `voting.enabled = true` means this board has a linked Ideas Board (`voting.name` = its display name)
- Show: `🚀 Help Center Deploy → ideas: 💡 Help Center Ideas`
- If user mentions an unknown word (e.g. "роадмапе", "маркетинг") — treat it as a board name filter, not a Ducalis entity type

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
- For creating boards or editing description/name/emoji -- see `references/board-settings.md`
