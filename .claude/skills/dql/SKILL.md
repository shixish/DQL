---
name: dql
description: "Query the Diffbot Knowledge Graph using DQL (Diffbot Query Language). Use when the user wants to search for organizations, people, or articles in the Diffbot KG. Triggers on: search diffbot, query knowledge graph, dql search, find companies in diffbot, diffbot lookup, kg search."
allowed-tools: Bash(curl*), Bash(grep*), Bash(python3*)
---

# Diffbot Knowledge Graph Search (DQL)

Query the Diffbot Knowledge Graph via the DQL API. Translate the user's plain-text request into a DQL query, execute it with curl, and display formatted results.

## Prerequisites: API Key

The API token must exist at `~/.diffbot/credentials`. Check with:

```bash
grep '^token=' ~/.diffbot/credentials
```

If missing, ask the user to run the one-time setup:

```bash
mkdir -p ~/.diffbot && chmod 700 ~/.diffbot
echo "token=YOUR_TOKEN_HERE" > ~/.diffbot/credentials
chmod 600 ~/.diffbot/credentials
```

Tokens are available at https://app.diffbot.com/get-started/

## Workflow

Always follow these steps in order:

### Step 1 — Read the token

```bash
TOKEN=$(grep '^token=' ~/.diffbot/credentials | cut -d= -f2 | tr -d '[:space:]')
```

Never echo or display the token value.

### Step 2 — Construct and validate the DQL query

Translate the user's natural language request into a DQL string, using the suggestion API to validate field names and values as you build it.

**Entity types**

Common types include `type:Organization`, `type:Person`, `type:Article`, and `type:Product`, but others exist. Use the suggestion API to discover available types:

```bash
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:"
```

`type:` is the minimum required field for any query. Default to `type:Organization` when the entity type is ambiguous.

**Operators**

| Operator | Syntax | Example |
|----------|--------|---------|
| Contains (string) | `field:"value"` | `name:"Diffbot"` |
| Regex | `re:field:"pattern"` | `re:name:"^Apple"` |
| Exact match | `strict:field:"value"` | `strict:name:"Apple Inc"` |
| Greater than | `field>N` | `nbEmployees>500` |
| Less than | `field<N` | `nbEmployees<10000` |
| AND (implicit) | space-separated | `type:Organization isPublic:true` |
| OR | `or(v1,v2)` | `industries:or("Software","Hardware")` |
| NOT | `not(condition)` | `not(isPublic:true)`, `not(has:parentCompany)` |
| Facet (aggregate) | `facet:field` | `facet:locations.city.name` |
| Sort ascending | `sortBy:field` | `sortBy:nbEmployees` |
| Sort descending | `revSortBy:field` | `revSortBy:nbEmployees` |

**Subquery syntax for nested fields**

Use `{}` to co-constrain multiple conditions on the same nested object:

```
type:Organization locations.{city.name:"San Francisco" isCurrent:true}
```

This ensures San Francisco is a *current* location, not that San Francisco is any location and some other location is current. Without `{}`, the two conditions are independent.

Not all fields support subqueries. The suggestion API indicates nestable fields by showing their type in brackets, e.g. `[Location].venue -> [String]`. Attempting `{}` on a non-nested field returns: `Nested expression over non-nested list field [...] is not allowed`.

**Using the suggestion API**

Use `https://kg.diffbot.com/kg/ac/dql?query={query}` (no token required) to validate field names and discover valid values while building the query. Iterate from partial queries:

```bash
# Discover fields available on Organization
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20"

# Validate a field name (e.g. check "ind" suggests "industries")
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20ind"

# Discover valid values for a field
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20industries:"
```

The response includes a `precision` field that shows the type of each suggestion, e.g. `[Location].venue -> [String]` — bracketed types indicate nestable fields that support subquery `{}` syntax.

Always validate unfamiliar field names and enum values via the suggestion API before executing the final query.

Before executing, display the final DQL query in a plain text code block so the user can copy or iterate on it.

### Step 3 — Execute the request

```bash
curl -s "https://kg.diffbot.com/kg/v3/dql?token=${TOKEN}&query=ENCODED_QUERY&size=10"
```

Default `size=10`. Increase up to `size=50` if the user asks for more results. For pagination append `&from=10`, `&from=20`, etc.

### Step 4 — Format and display results

The response structure:
- `hits` — total matching entity count (integer)
- `data[]` — array of result objects, each with an `entity` key

Pipe the curl output into an inline Python script to parse and display results. The response structure:
- `hits` — total match count
- `data[].entity` — the entity objects

Write an appropriate formatter for the entity type being queried. For facet queries, aggregated counts are in a `facets` key alongside `data`.

After displaying results, offer to fetch the next page or refine the query.

## Error handling

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| HTTP 401 | Invalid or missing token | Check `~/.diffbot/credentials` |
| HTTP 400 | Malformed DQL | Check field names and operators |
| `hits.total = 0` | Query too restrictive | Broaden filters or check spelling |
| `curl: (6) Could not resolve host` | No network | Check connectivity |
| Empty `~/.diffbot/credentials` | Setup not done | Run the setup commands above |
