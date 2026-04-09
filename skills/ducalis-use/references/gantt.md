## Gantt — schedule, plan, and track task timelines

### Today is {{TODAY}}

### Core concept

Board Gantt shows tasks from one board on a timeline. Two pools:
- **On Gantt** — tasks added to the Gantt chart (have position, may or may not have dates)
- **Unscheduled** — rest of the board tasks (not on Gantt yet)

`scheduled` field: `true` if task has start/end dates set, `false` if on Gantt but no dates yet. A task can be on Gantt without dates — it's waiting to be scheduled.

Descriptions are NOT needed for Gantt — focus on: name, id, score, position, dates.

### STEP 1 — Always read current Gantt first

Before ANY Gantt operation, read what's already there:
```
read_ducalis({ resource: "gantt", board_uuid, include: ["position", "scheduled", "dependencies"], limit: 50 })
```
This returns ALL tasks on Gantt (with AND without dates). Use `scheduled` to distinguish.

**Batch fetch specific entries** (when you have issue IDs):
```
read_ducalis({ resource: "gantt", board_uuid, issue_ids: [id1, id2, ...], include: ["position", "scheduled"] })
```

**Fetch full issue data for Gantt tasks** (score, criteria, assignee):
```
read_ducalis({ resource: "issues", board_uuid, issue_ids: [id1, id2, ...], include: ["score", "criteria_scores", "assignee"] })
```

To see what's NOT on Gantt (unscheduled pool):
```
read_ducalis({ resource: "issues", board_uuid, sort_by: "score", sort_order: "desc", include: ["score"], limit: 20 })
```
Compare issue IDs with gantt results — issues NOT in gantt are unscheduled.

### Adding tasks to Gantt with dates (combined)

**plan_gantt** — Add tasks with position, color, AND dates in ONE action
- Required: `board_uuid`, `entries[]` — each with `issue_id`, `position` (number), `start_date` (YYYY-MM-DD), `end_date` (YYYY-MM-DD), optional `color` (hex), `issue_name`
- **Use this when you have both positions and dates** (e.g. capacity planning, sprint planning)
- **Color strategy**: assign colors by topic/theme. Related tasks (same area, blockers) get same color. Different themes get different colors. Palette: `#4A90D9` `#E67E22` `#27AE60` `#9B59B6` `#E74C3C` `#1ABC9C` `#F39C12` `#2C3E50`

```
write_ducalis({ action: "plan_gantt", params: { board_uuid, entries: [
  { issue_id: 123, position: 1, color: "#4A90D9", start_date: "2026-04-03", end_date: "2026-04-06", issue_name: "Task A" }
] }, confirm: false })
```

### Adding tasks to Gantt (position only)

**set_board_gantt** — Add tasks to Gantt with position (and optional color), no dates
- Required: `board_uuid`, `entries[]` — each with `issue_id` + `position` (number) and/or `color` (hex)

### Setting dates only

**set_gantt_dates** — Bulk set start/end dates for issues already on Gantt
- Required: `entries[]` — each with `issue_id`, `start_date` (YYYY-MM-DD or null), `end_date` (YYYY-MM-DD or null)
- Pass `null` to clear a date

**delete_gantt_dates** — Remove all dates from one issue
- Required: `issue_id`

### Removing from Gantt

**remove_board_gantt** — Bulk remove tasks from Gantt (returns to Unscheduled pool)
- Required: `board_uuid`, `issue_ids[]`

Use for cleanup: "remove all tasks without dates from Gantt", "clear the Gantt", etc.

### Dependencies

Two types of connections between tasks:
- `link` = **Visual Link** — visual connection only, no scheduling constraint (dashed line)
- `block` = **Finish to Start** — successor starts after predecessor ends + workday offset (solid arrow)

Use `link_issues` / `unlink_issues` with `board_uuid`:
```
write_ducalis({ action: "link_issues", params: { board_uuid, issue_id, linked_issue_id, type: "block" }, confirm: true })
```

### Date handling rules

**Format: YYYY-MM-DD only** (e.g. `"2026-04-03"`). NEVER use DD.MM, DD.MM.YYYY, or other formats — the API will reject them. Use YYYY-MM-DD in tables, previews, and API calls consistently.

**Weekend skipping**: Only workdays (Mon-Fri) count.
- A task ending Friday: next task starts Monday
- A 1-day task on Friday: start=Friday, end=Friday
- A 2-day task starting Friday: start=Friday, end=Monday

**Sequential scheduling ("stairs")** — when placing tasks one after another:
```
cursor = start_date (must be a weekday)
for each task:
    task.start_date = cursor
    task.end_date   = cursor + (working_days - 1), skipping weekends
    cursor          = next_workday(task.end_date)  // +1 day, skip Sat/Sun
```

Worked example (start = 2026-04-03 Friday):
| # | Days | Start | End | Why |
|---|------|-------|-----|-----|
| 1 | 2д | 2026-04-03 | 2026-04-06 | Fri + Mon = 2 workdays |
| 2 | 1д | 2026-04-07 | 2026-04-07 | Tue (next workday after Mon) |
| 3 | 5д | 2026-04-08 | 2026-04-14 | Wed-Fri + Mon-Tue = 5 workdays |

**Key rule: each task's start = previous task's end + 1 workday. No gaps, no overlaps.**

**Table = API dates**: Dates shown in plan tables MUST be the EXACT same dates sent to `plan_gantt` / `set_gantt_dates`. Do NOT recompute dates when making the API call.

### Planning rules

- **Finish to Start**: blocked task's start_date MUST be AFTER blocker's end_date
- **Always read first**: never set dates without checking current Gantt state
- **Combined action**: use `plan_gantt` when you have positions AND dates — one action, one preview, one confirmation
- **Separate actions**: use `set_board_gantt` (position only) + `set_gantt_dates` (dates only) when modifying just one aspect
- After changes: `[timeline](/board/{uuid}/gantt)`

For "open Gantt": `[Open Gantt](/board/{uuid}/gantt)`. Link tasks as `[Task Name](/issue/{id})`.
