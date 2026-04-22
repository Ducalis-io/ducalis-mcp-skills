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

Common fields (defaults): `id`, `name`, `email`, `company` (string), `votes_count`. When `query` is supplied, extra fuzzy-resolver fields `score` (0..1) and `match_reason` are attached so the agent can see why each candidate matched.
Available via `include`: `ideas_count`, `comments_count`, `plan_name`, `paying_now`, `mrr`, `total_spend`, `annual_plan`, `last_activity`, `signup_date`, `subscription_start`, `users`, `voted_ideas`, `trackers`, `subscribed_boards`, `accessible_boards`.

**Fuzzy search with cyrillic:** `/storage/voting-users?query=…` on the backend runs ILIKE and does NOT transliterate — a bare `query: "Камиль"` would miss `Kamil Samigullin` entirely. The `voting_users` resource handles this client-side: it generates ru↔en variants, fires parallel probes, unions results, and re-ranks with token + company overlap + activity prior. The composite form `"<Name> из/from <Company>"` is parsed: the company part is a strong ranking boost.

**Single-call fuzzy search — ALWAYS via `query`, never via multiple `where` lookups.** One `query` string covers all name/email/company variants in a single tool call. Do NOT split it into 2–3 separate `where` calls (per company-name variant, per translit). The resolver internally generates ru↔en transliteration, punctuation variants ("Т-Банк" ↔ "Тбанк" ↔ "Tbank"), and strips Russian grammatical endings ("Т-Банка" → "Т-Банк"). If 0 hits — the voter really doesn't exist, offer `create_voting_user`; don't retry with different spellings.

**Search by name/email (fuzzy):**
```
read_ducalis({ resource: "voting_users", query: "<voter name>" })
read_ducalis({ resource: "voting_users", query: "<voter name> из <Company>" })
read_ducalis({ resource: "voting_users", query: "<voter@example.com>" })
```
Output is ranked: top candidate first, each item includes `score` and `match_reason`. Exact-email match always scores 1.0. Prefer the top candidate unless `score` < 0.6 or two results are near-tied, in which case ask the user.

Or via `where` for strict filters:
```
read_ducalis({ resource: "voting_users", where: { field: "email", op: "contains", value: "@example.com" } })
```

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
