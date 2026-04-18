## Voters & Companies — read patterns

Voters (`voting_users`) and companies (`voting_companies`) are the **customer-context layer** for ideas: who's asking, from where, what plan, how much they pay. Use them to segment feedback, find customer-specific demand, or discover who's behind an idea.

### When to use

- "What is company X asking for?" / "ideas from Acme"
- "Show ideas from enterprise / paying customers"
- "Who voted for idea N?"
- "Top voters this quarter"
- Pre-create lookup: find a voter to attribute a new idea to (see `idea-write.md`)

### Resource: `voting_users`

```
read_ducalis({ resource: "voting_users", limit: 30, sort_by: "votes_count", sort_order: "desc" })
```

Common fields (defaults): `id`, `name`, `email`, `company` (string), `votes_count`.
Available via `include`: `ideas_count`, `comments_count`, `plan_name`, `paying_now`, `mrr`, `total_spend`, `annual_plan`, `last_activity`, `signup_date`, `subscribed_boards`, `accessible_boards`.

**Search by name/email** — server-side via Ducalis suggests:
```
read_ducalis({ resource: "voting_users", query: "John" })
read_ducalis({ resource: "voting_users", query: "john@example.com" })
```
Or via `where`:
```
read_ducalis({ resource: "voting_users", where: { field: "email", op: "contains", value: "@example.com" } })
```
(both routes call the same `/rest/voting-users/suggests` endpoint).

**Filter paying / segment** — the `vcf_*` fields are flattened from the user's company and surfaced as plain fields on the voter:
```
read_ducalis({
  resource: "voting_users",
  where: [
    { field: "paying_now", op: "eq", value: true },
    { field: "plan_name", op: "eq", value: "Business" }
  ],
  include: ["plan_name", "mrr", "paying_now"],
  sort_by: "mrr", sort_order: "desc"
})
```

**Per-voter activity** (singular) — pulls authored/voted ideas in one call:
```
read_ducalis({
  resource: "voting_user",
  issue_id: 8,
  include: ["authored_idea_ids", "voted_idea_ids", "commented_idea_ids"]
})
```

### Resource: `voting_companies`

```
read_ducalis({ resource: "voting_companies", sort_by: "ideas_count", sort_order: "desc" })
```

Defaults: `id`, `name`, `users_count`, `ideas_count`, `votes_count`. Same `vcf_*` plan/paying/mrr fields as voters.

**Per-company detail**:
```
read_ducalis({
  resource: "voting_company",
  issue_id: 9067,
  include: ["users", "authored_idea_ids", "voted_idea_ids"]
})
```
Returns the list of users belonging to the company plus the union of ideas they authored or voted on.

### Filtering ideas by voter / company

`ideas` resource accepts four optional filters that pre-narrow the result set BEFORE field extraction. Combine with `where` for further narrowing.

| param | effect |
|---|---|
| `voter_id` | Ideas where this voter is author OR has voted |
| `voter_email` | Same, but resolves email → voter id automatically (throws if not found) |
| `company_id` | Ideas where any user of the company is author OR has voted |
| `company_name` | Same, by exact case-insensitive company name |

```
// "What did this voter ask for or vote on?"
read_ducalis({ resource: "ideas", board_uuid, voter_email: "<voter@example.com>" })

// "What does <Customer> want on this board?"
read_ducalis({ resource: "ideas", board_uuid, company_name: "<Customer>" })

// Combine: ideas voted by a specific company in status Inbox
read_ducalis({
  resource: "ideas",
  board_uuid,
  company_name: "<Customer>",
  where: { field: "status", op: "eq", value: "Inbox" }
})
```

### Voter list on a single idea

`idea` singular auto-includes `voting_users` and `companies`. To see only counts/emails:

```
read_ducalis({
  resource: "idea",
  issue_id: 12345,
  include: ["voter_count", "voter_emails", "company_names"]
})
```

### Idea voting dynamics

```
read_ducalis({ resource: "idea", issue_id: 12345, include: ["stats"] })
```
Pulls `/rest/boards/{uuid}/ideas/{id}/stats` (votes_total, voters_count, companies_count, optional time series).

### Gotchas

- `company` on a voter is a **string** (`company_name`), not an id. To filter the idea set by a real company, use `voting_companies` to resolve to `id` first OR use `company_name` on `ideas`.
- Voters from `/storage/voting-users` already include rich `vcf_*` (plan/mrr/paying). Don't ask the user to fetch a separate "company plan" call.
- Only authored+voted are joined for `voter_id`/`company_id` filters. "Commented" ideas don't count toward the filter (use the singular `voting_user` + `commented_idea_ids` include for that).
