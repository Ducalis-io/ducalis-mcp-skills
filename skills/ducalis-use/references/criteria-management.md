## Criteria Management — design, link, configure, evaluators, facilitators

### What is a criterion

A **Criterion** is a principle for evaluating issue priority. It's a **dimension of evaluation** — like "Business Impact", "Development Effort", "Revenue Potential", "Technical Risk", "Urgency". It is NOT a task, feature, or use case.

**Types:**
- **Value** — benefit dimension. Higher score = higher priority. Examples: Impact, Reach, Revenue, Strategic Fit
- **Effort** — cost dimension. Higher score = pushes priority DOWN. Examples: Dev Time, Complexity, Risk
- **Ignore** — excluded from Total Score formula

**Common frameworks** (for inspiration, not to copy blindly):
- RICE: Reach, Impact, Confidence, Effort
- ICE: Impact, Confidence, Ease
- WSJF: Business Value, Time Criticality, Risk Reduction, Job Size
- Value vs Effort: single Value criterion + single Effort criterion

**Scales:** Range (1-5, 1-10), Series (1,2,3,5,8,13 — Fibonacci), Percent (0-100%), Positive (any positive number)

**Description:** Each criterion MUST have a description explaining what it measures and what each scale value means. Without descriptions, evaluators guess and scores become inconsistent.

### Criteria design algorithm

When user asks to create or redesign criteria, follow this algorithm. **Never design criteria in a vacuum.**

#### Gather context (always do all 3 steps)

**Step 1 — Board context:**
```
read_ducalis({ resource: "board", board_uuid: "...", include: ["description", "criteria_full"] })
```
→ Board name, description (strategic goal), existing criteria

**Step 2 — Backlog landscape (50 task titles):**
```
read_ducalis({ resource: "issues", board_uuid: "...", include: ["name"], limit: 50 })
```
→ Scan titles to understand what kinds of tasks live in this board

**Step 3 — Sample task details (~10 tasks):**
Pick ~10 representative tasks from different areas and fetch their descriptions:
```
read_ducalis({ resource: "issues", board_uuid: "...", issue_ids: [...], include: ["description"] })
```
→ Deeper understanding of task granularity and domain

#### Assess context sufficiency

After gathering, evaluate what you have:

| Signal | Where to find it |
|--------|-----------------|
| Board goal / strategy | Board description, board name |
| Domain / product area | Task titles (patterns, keywords, clusters) |
| Task granularity | Task descriptions (are they features? bugs? epics?) |
| User's priorities | User's message (what matters to them, what problems they face) |
| Existing evaluation system | Current criteria (names, scales, descriptions) |

**Context levels:**

**Rich** (board description + coherent tasks + user explains their goals):
→ Proceed to propose criteria directly. You have enough to identify relevant evaluation dimensions.

**Medium** (tasks exist but no board description, or user request is vague like "make criteria"):
→ Summarize what you see in the backlog (themes, clusters, task types). Propose criteria with a caveat: "Based on the tasks I see, here's what I'd suggest — but tell me more about your goals if you want a more tailored system." Ask 1-2 focused questions:
  - "What's the main goal of this board — shipping features, reducing tech debt, triaging incoming ideas?"
  - "Who will be evaluating — the whole team, or specific experts per dimension?"

**Low** (< 5 tasks, no description, generic request like "add criteria"):
→ Do NOT propose criteria yet. Ask the user:
  - "What is this board for? What kind of decisions should criteria help you make?"
  - "Can you describe 2-3 example tasks and what makes one more important than another?"
  - "Are you following a framework (RICE, ICE, WSJF) or building custom?"

#### Evaluate existing criteria

If the board already has criteria, assess each one against the tasks you read:
- Does this criterion make sense for THIS board's tasks?
- Is the scale appropriate for the task granularity?
- Is the description clear enough for evaluators?
- Are there obvious gaps (e.g. only Value criteria, no Effort)?

Then **ASK the user** before making any changes. Present your assessment and offer clear options:
- **Option A: Extend** — keep existing criteria, add new ones to cover gaps
- **Option B: Replace** — unlink criteria that don't fit, link/create better ones
- **Option C: Adjust** — keep the same criteria but update their weights, descriptions, scales, or evaluator assignments

Do NOT jump to creating new criteria without this conversation. The user may want to keep what they have and just tweak it.

#### Propose criteria

When proposing, for each criterion provide:
- **Name** — short, clear, universal (applies to ALL tasks in the board)
- **Type** — value or effort, with reasoning
- **Scale** — which type and values, with reasoning
- **Description** — what it measures + what each scale level means (this is critical for evaluator alignment)
- **Weight suggestion** — relative importance vs other criteria

**IMPORTANT:** If the user describes use cases, product scenarios, or features in their message — these are usually CONTEXT about what the board contains (or what their product does), NOT instructions to create criteria named after those cases. Criteria should be universal evaluation dimensions that apply across ALL tasks in the board.

For example, if user says "we have tasks about AI search, deduplication, and triage" — the criteria should be something like "User Impact", "Revenue Potential", "Dev Effort", "Strategic Alignment" — dimensions that let you compare and rank ALL those tasks against each other. NOT criteria called "AI Search Score" or "Deduplication Risk".

### Open criteria settings — ALWAYS for write intent

When the user wants to **modify** criteria (add, remove, update, reconfigure) — not just read for a side task — ALWAYS include a link to the criteria settings page early in the response:

`[Open criteria settings](/board/{uuid}/summary?dialog=/settings/board/{uuid}/criteria)`

Include this link:
- At the start of any criteria management conversation (when board_uuid is known)
- After executing write actions so the user can verify
- When suggesting changes that require admin-only operations (global create/edit)

### Read current setup

**Board criteria with full settings:**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["criteria_full"] })`
→ Per criterion: name, type, description, scale, coefficient, required, votes_lifetime, reset_votes, assigned_users, facilitators

**Board members (for evaluator/facilitator assignment):**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["members_full"] })`

### Before adding criterion — SEARCH FIRST

Criteria are **workspace-scoped** — same criterion can be linked to multiple boards.
1. Read `criteria_full` on the target board AND other boards (`resource: "boards"` without board_uuid for all)
2. If matching criterion exists → suggest reuse via `link_criterion`
3. Only if nothing matches → inform user that creating new criteria requires admin access in Ducalis settings

### Write actions

| Action | Purpose | Key params |
|--------|---------|------------|
| `link_criterion` | Link existing criterion to board | `board_uuid`, `criterion_id` |
| `unlink_criterion` | Remove criterion from board (global preserved) | `board_uuid`, `criterion_id` |
| `update_board_criterion` | Change weight, required, votes lifetime | `board_uuid`, `criterion_id` + `coefficient`, `required`, `votes_lifetime`, `reset_votes` |
| `shift_criterion` | Reorder criterion position | `board_uuid`, `criterion_id`, `position` |
| `add_evaluator` | Assign user as evaluator | `board_uuid`, `criterion_id`, `user_id` |
| `remove_evaluator` | Remove evaluator | `board_uuid`, `criterion_id`, `user_id` |
| `add_facilitator` | Assign user as facilitator (can override scores) | `board_uuid`, `criterion_id`, `user_id` |
| `remove_facilitator` | Remove facilitator | `board_uuid`, `criterion_id`, `user_id` |

All accept optional `criterion_name`, `user_name` for readable preview.

### Workflow: "design criteria for this board"

1. Show criteria settings link: `[Open criteria settings](/board/{uuid}/summary?dialog=/settings/board/{uuid}/criteria)`
2. Gather context: board info + 50 titles + ~10 descriptions (Steps 1-3)
3. Assess context sufficiency: Rich → propose; Medium → propose + ask; Low → ask first
4. If existing criteria → evaluate them, present assessment, ask: extend / replace / adjust?
5. Propose criteria with: name, type, scale, description per level, weight suggestion
6. Search existing workspace criteria for reuse before creating new
7. For repetitive operations (assign evaluator to many criteria): use `batch` action

### Gotchas

- **Cannot create/edit/delete global criteria** — only link existing ones to boards. When user needs this, direct them to the criteria settings page (link above)
- Criteria are EVALUATION DIMENSIONS, not task categories — they must apply across all tasks
- `votes_lifetime > 0` automatically enables `reset_votes`
- After `unlink_criterion`: evaluator/facilitator assignments removed, votes preserved in global criterion
- Use `batch` meta-action for multiple evaluator assignments
