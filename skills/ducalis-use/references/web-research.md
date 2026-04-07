## Web Research — search the internet on explicit user request

### CRITICAL RULES

1. **NEVER use proactively** — ONLY when user explicitly asks to search the web,
   read a URL, or research something online. If the task can be solved with
   Ducalis data alone — do NOT go to the web.
2. **ALWAYS confirm before calling** — describe what you plan to search/read
   and WHY, then wait for user approval before making the tool call.
3. **Expensive operation** — each call costs API credits. Be frugal:
   basic search_depth, limit results to 5, extract only needed URLs.

### Confirmation flow

1. User asks to search/read something from the web
2. You respond: "Я поищу в интернете: [что именно]. Это поможет [зачем]. Искать?"
3. User confirms → call `web_research`
4. Present results with source URLs

### Patterns

**Quick web search:**
```
web_research({ action: "search", query: "RICE scoring framework best practices" })
```

**Search within specific domains:**
```
web_research({ action: "search", query: "pricing", include_domains: ["competitor.com"] })
```

**Read a specific page:**
```
web_research({ action: "extract", urls: ["https://docs.example.com/api"] })
```

**Deep research (expensive — confirm twice):**
```
web_research({ action: "research", query: "Compare RICE vs ICE vs WSJF prioritization" })
```

### When NOT to use

- Task can be answered from Ducalis data (boards, issues, scores)
- General knowledge questions the model already knows
- User didn't explicitly mention internet/web/search/URL
- User asks to "find" or "search" issues — that's `read_ducalis`, not web search
