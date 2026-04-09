## Criteria Management — design, link, configure, evaluators, facilitators

### Admin role requirement

Creating, editing, and deleting **global** criteria requires **admin** role.
Check `## Current Context` for the user's role.

- **Admin**: can create new criteria (`create_criterion`), edit (`update_criterion`), delete (`delete_criterion`), plus all board-scoped operations
- **Non-admin**: can only use board-scoped operations (link existing criteria, configure weight/evaluators). For new criteria, tell the user: "Creating new criteria requires admin role. Ask your workspace admin or create manually in [criteria settings](/board/{uuid}/summary?dialog=/settings/board/{uuid}/criteria)."

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

**Best practice: Fibonacci series (1,2,3,5,8,13)** — default scale for most criteria. Increasing gaps force meaningful distinctions. Only use range/percent when there's a specific reason (percent for confidence, range 1-3 for binary-ish decisions).

**Description:** Each criterion MUST have a description explaining what it measures and what each scale value means. Without descriptions, evaluators guess and scores become inconsistent.

### Criteria design algorithm

When user asks to create or redesign criteria, follow this algorithm. **Never design criteria in a vacuum.**

#### Gather context

**Round 1 — Board + issues (parallel, ONE round):**
Make BOTH calls together in a single round:
```
read_ducalis({ resource: "board", board_uuid: "...", include: ["description", "criteria_full"] })
read_ducalis({ resource: "issues", board_uuid: "...", include: ["name"], limit: 50 })
```
Board name, description, existing criteria + issue titles. Check "Total issues on board: N" in issues response.

**After Round 1 — decide next step:**
- **0 issues**: STOP gathering. Go to "Assess context sufficiency" with board name + description + existing criteria.
- **1-5 issues**: All titles visible. STOP gathering. Go to "Assess context sufficiency".
- **6+ issues**: ONE more call — pick ~10 representative tasks from different areas:
  `read_ducalis({ resource: "issues", board_uuid: "...", issue_ids: [...], include: ["description"] })`

**IMPORTANT:** Do NOT make any other calls during context gathering. No `read_ducalis({ resource: "boards" })`, no dictionaries, no duplicate calls. Maximum 2-3 total calls for this phase.

#### Assess context sufficiency

| Signal | Where to find it |
|--------|-----------------|
| Board goal / strategy | Board description, board name |
| Domain / product area | Task titles (patterns, keywords) |
| Task granularity | Task descriptions (features? bugs? epics?) |
| User's priorities | User's message |
| Existing evaluation system | Current criteria (names, scales, descriptions) |

**Context levels:**

**Rich** (board description + 6+ coherent tasks + user explains goals):
Proceed to propose criteria directly.

**Medium** (board has description but few/no tasks, OR tasks exist but no description, OR vague request like "make criteria"):
Propose criteria based on available info with a caveat: "Based on [board description / the N tasks I see], here's what I'd suggest. Tell me more about your goals for a more tailored system." Ask 1-2 focused questions:
  - "What's the main goal of this board — shipping features, reducing tech debt, triaging incoming ideas?"
  - "Who will be evaluating — the whole team, or specific experts per dimension?"

**Low** (no tasks AND no description, generic request like "add criteria"):
Do NOT propose criteria yet. Ask the user:
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
Returns per criterion: name, type, description, scale, coefficient, required, votes_lifetime, reset_votes, assigned_users, facilitators

**Board members (for evaluator/facilitator assignment):**
`read_ducalis({ resource: "board", board_uuid: "...", include: ["members_full"] })`

### Before adding criterion — SEARCH FIRST

Criteria are **workspace-scoped** — same criterion can be linked to multiple boards.

**When to search:** After user approves your proposal, BEFORE executing writes:
1. Fetch ALL workspace criteria: `read_ducalis({ resource: "criteria" })`
   Returns compact list: `[type] Name (id: N)` for every criterion in the workspace.
2. Compare your proposed criteria against workspace list by name/type similarity.
3. If matching criterion exists -- suggest reuse: `link_criterion` + `update_board_criterion` if weight differs.
4. Only if nothing matches AND user is admin -- create via `create_and_link_criterion` (combines create + link + configure in one action).
5. If user is not admin -- direct to criteria settings page.

### Write actions

**Global (admin only):**

| Action | Purpose | Key params |
|--------|---------|------------|
| `create_criterion` | Create new global criterion | `name`, `type` (value/effort), `scale_type` (range/series/percent/positive), `scale_min`/`scale_max` or `scale_series`, `description` (HTML) |
| `create_and_link_criterion` | Create + link to board + configure in one step | `board_uuid`, `name`, `type`, `scale_type`, scale params, `description`, `coefficient`, `required` |
| `update_criterion` | Edit global criterion (affects all boards) | `criterion_id` + fields to change |
| `delete_criterion` | Delete global criterion (PERMANENT) | `criterion_id` |

**Board-scoped (all roles):**

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
2. Round 1: board info + issues (parallel, 2 calls). Check "Total issues on board: N"
3. If 6+ issues: Round 2 (~10 task descriptions). Otherwise STOP gathering
4. Assess context: Rich: propose. Medium: propose + ask. Low: ask first
5. If existing criteria: evaluate, ask user: extend / replace / adjust?
6. Propose criteria with: name, type, scale (Fibonacci by default), description per level, weight
7. After user approves: fetch workspace criteria `read_ducalis({ resource: "criteria" })`
8. Reusable: `link_criterion` + `update_board_criterion`
9. New: `batch` + `create_and_link_criterion` (one tool call for all new criteria)
10. After criteria linked: guide evaluator assignment (see below)

### Evaluator assignment flow

After criteria are linked, guide evaluator setup:

1. Read current state:
   `read_ducalis({ resource: "board", board_uuid: "...", include: ["criteria_full", "members_full"] })`
2. Identify criteria with NO evaluators — highlight as warning:
   "These criteria have no evaluators. Issues cannot be fully scored until evaluators are set."
3. Suggest assignments based on criterion type:
   - Technical criteria (dev effort, complexity): engineering members
   - Business criteria (revenue, strategic fit): product/business members
   - If unsure, ask: "Who on your team is best positioned to evaluate [criterion]?"
4. If board has no suitable members:
   `[Invite members](/board/{uuid}/summary?dialog=/settings/board/{uuid}/members)`
5. Use `batch` + `add_evaluator` for multiple assignments

### Gotchas

- **Global criteria operations** (create, update, delete) require **admin** role — check context before calling
- Criteria are EVALUATION DIMENSIONS, not task categories — they must apply across all tasks
- `votes_lifetime > 0` automatically enables `reset_votes`
- After `unlink_criterion`: evaluator/facilitator assignments removed, votes preserved in global criterion
- Use `batch` meta-action for multiple evaluator assignments
