---
name: dql
description: "Query the Diffbot Knowledge Graph using DQL (Diffbot Query Language). Use when the user wants to search for organizations, people, or articles in the Diffbot KG. Triggers on: search diffbot, query knowledge graph, dql search, find companies in diffbot, diffbot lookup, kg search."
allowed-tools: Bash(curl*), Bash(grep*), Bash(python3*), Bash(jq*)
---

# Diffbot Knowledge Graph Search (DQL)

Query the Diffbot Knowledge Graph via the DQL API. Translate the user's plain-text request into a DQL query, execute it with curl, and display formatted results.

## Workflow

Always follow these steps in order:

### Step 1 — Initialize credentials and ontology cache

**Step 1a** — Create `~/.diffbot/` and refresh the ontology cache (no token required):

```bash
mkdir -p ~/.diffbot && curl -s "https://kg.diffbot.com/kg/ontology" > ~/.diffbot/ontology.json
```

**Step 1b** — Read the API token:

```bash
TOKEN=$(grep '^token=' ~/.diffbot/credentials | cut -d= -f2 | tr -d '[:space:]')
```

Never echo or display the token value.

If `~/.diffbot/credentials` is missing, the directory already exists from Step 1a. Ask the user to run:

```bash
echo "token=YOUR_TOKEN_HERE" > ~/.diffbot/credentials && chmod 600 ~/.diffbot/credentials
```

Tokens are available at https://app.diffbot.com/get-started/

### Step 2 — Construct and validate the DQL query

Translate the user's natural language request into a DQL string. Use the ontology cache (Step 2a) and suggestion API (Step 2b) as needed while building the query.

#### Step 2a — Query the ontology cache

Use `jq` against `~/.diffbot/ontology.json` to look up field names, types, and valid values before writing DQL. This avoids typos and ensures enum values are exact.

```bash
# All entity type names
jq '.types | keys[]' ~/.diffbot/ontology.json

# All queryable (non-deprecated) fields for a type, with value type and list flag
jq '.types.Organization.fields | to_entries[]
    | select(.value.isDeprecated==false)
    | "\(.key): \(.value.type)\(if .value.isList then " []" else "" end)"' \
    ~/.diffbot/ontology.json

# Fields for a composite type (e.g. Employment, Location)
jq '.composites.Employment.fields | keys[]' ~/.diffbot/ontology.json

# Valid values for an enum
jq '.enums.Industry.values[]' ~/.diffbot/ontology.json

# Type hierarchy (which parent types contribute inherited fields)
jq '.types.Organization.typeHierarchy' ~/.diffbot/ontology.json

# Top-level taxonomy category names
jq '.taxonomies.IndustryCategory.categories[].name' ~/.diffbot/ontology.json
```

**Ontology schema reference**

The cached file has these top-level keys:

```
metadata    — kg-version, binary-version, generated timestamp
types       — 61 entity types (Organization, Person, Article, Product, …)
              Each entry: { name, typeHierarchy[], fields{}, documented }
composites  — 54 sub-object types used as field values (e.g. Location, Employment)
              Same shape as types
enums       — 26 enumerations; each: { name, values[] }
taxonomies  — 5 hierarchical category trees (IndustryCategory, EmploymentCategory, …)
              Each entry: { categories[{ name, info?, children? }] }
dev-notes   — human-readable description string
```

Each field definition inside `types` and `composites` has:

```
name, description, type (String/Integer/composite name/…),
isList, isPrimitive, isComposite, isEntity, isEnum, isDeprecated,
isFact, documented, tracksFirstSeen,
leType[]   — linked entity type names (when isEntity: true)
taxonomy   — taxonomy name (when field draws from a taxonomy)
```

#### Step 2b — Suggestion API for partial-string autocomplete

Use `https://kg.diffbot.com/kg/ac/dql?query={query}` (no token required) to autocomplete partial field names or discover valid enum values you haven't looked up yet:

```bash
# Discover fields available on Organization
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20"

# Validate a partial field name (e.g. "ind" → "industries")
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20ind"

# Discover valid values for a field
curl -s "https://kg.diffbot.com/kg/ac/dql?query=type:Organization%20industries:"
```

The response includes a `precision` field showing each suggestion's type, e.g. `[Location].venue -> [String]` — bracketed types indicate nestable fields that support subquery `{}` syntax.

Prefer the ontology cache for bulk field/enum discovery. Use the suggestion API for partial-string autocomplete or when exploring an unfamiliar field's values.

**Entity types**

`type:` is the minimum required field for any query. Common types: `type:Organization`, `type:Person`, `type:Article`, `type:Product`. Default to `type:Organization` when ambiguous.

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

Pipe the curl output into an inline Python script to parse and display results. Write an appropriate formatter for the entity type being queried. For facet queries, aggregated counts are in a `facets` key alongside `data`.

After displaying results, offer to fetch the next page or refine the query.

## Error handling

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| HTTP 401 | Invalid or missing token | Check `~/.diffbot/credentials` |
| HTTP 400 | Malformed DQL | Check field names and operators |
| `hits.total = 0` | Query too restrictive | Broaden filters or check spelling |
| `curl: (6) Could not resolve host` | No network | Check connectivity |
| Empty `~/.diffbot/credentials` | Setup not done | Run the setup commands above |
