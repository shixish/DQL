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

```bash
mkdir -p ~/.diffbot && curl -s "https://kg.diffbot.com/kg/ontology" > ~/.diffbot/ontology.json && TOKEN=$(grep '^token=' ~/.diffbot/credentials | cut -d= -f2 | tr -d '[:space:]')
```

Never echo or display the token value.

If `~/.diffbot/credentials` is missing, the directory already exists. Ask the user to run:

```bash
echo "token=YOUR_TOKEN_HERE" > ~/.diffbot/credentials && chmod 600 ~/.diffbot/credentials
```

Tokens are available at https://app.diffbot.com/get-started/

### Step 2 — Construct and validate the DQL query

Translate the user's natural language request into a DQL string. `type:` is the minimum required field for any query — default to `type:Organization` when ambiguous. For well-known entity types (Organization, Person, Article, Product), construct the query directly. For unfamiliar types or fields, use `jq` against `~/.diffbot/ontology.json` to look up field names, types, and valid values before writing DQL.

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

**Article tags and categories**

`type:Article` queries have special handling for tags (entity mentions with sentiment) and categories (IAB/Diffbot taxonomy labels). Tags can be searched by entity URI, label, or type; categories by ID or name. Both support `OR()`, subquery `{}` syntax, and faceting. See [Article Tags and Categories](https://docs.diffbot.com/docs/article-tags-and-categories.md).

**Operators**

| Operator             | Syntax                 | Example                                                  |
| -------------------- | ---------------------- | -------------------------------------------------------- |
| Contains (string)    | `field:"value"`        | `name:"Diffbot"`                                         |
| Regex                | `re:field:"pattern"`   | `re:name:"^Apple"`                                       |
| Exact match          | `strict:field:"value"` | `strict:name:"Apple Inc"`                                |
| Greater than         | `field>N`              | `nbEmployees>500`                                        |
| Less than            | `field<N`              | `nbEmployees<10000`                                      |
| Not equals           | `field!=value`         | `gender!="MALE"`                                         |
| Max                  | `max:field:N`          | `max:capitalization.value:1000000`                       |
| Min                  | `min:field:N`          | `min:capitalization.value:1000000`                       |
| Range                | `range:field:N-M`      | `range:nbEmployees:10-100`                               |
| AND (implicit)       | space-separated        | `type:Organization isPublic:true`                        |
| OR                   | `or(v1,v2)`            | `industries:or("Software","Hardware")`                   |
| NOT                  | `not(condition)`       | `not(isPublic:true)`, `not(has:parentCompany)`           |
| Near (proximity)     | `near(name:"Place")`   | `near(name:"San Francisco", 10mi)`                       |
| Similar to           | `similarTo(…)`         | `similarTo(type:Organization homepageUri:"walmart.com")` |
| Has (field exists)   | `has:field`            | `has:sicClassification`                                  |
| Get (include fields) | `get:field`            | `has:subsidiaries get:subsidiaries`                      |
| Get (exclude fields) | `get:!field`           | `get:!nbEmployeesMax,!phoneNumbers`                      |
| Facet (aggregate)    | `facet:field`          | `facet:locations.city.name`                              |
| Facet with ranges    | `facet[a:b,b:c]:field` | `facet[100:500,500:1000]:nbEmployees`                    |
| Facet with values    | `facet["a","b"]:field` | `facet["Austin","Seattle"]:locations.city.name`          |
| Sort ascending       | `sortBy:field`         | `sortBy:nbEmployees`                                     |
| Sort descending      | `revSortBy:field`      | `revSortBy:nbEmployees`                                  |

**similarTo operator** _(experimental — Organization only)_

Finds entities similar to a given entity. See [similarTo Operator](https://docs.diffbot.com/docs/similarto-operator-1.md) for full syntax and examples.

**near operator**

Finds entities within a given distance of a Place (default 15km). Distance can be specified in `mi` or `km`. Only the highest-ranked matching place is used.

**get operator**

`get:field` includes only specified fields (and their descendants) in results; `get:!field` excludes them. Multiple fields are comma-separated. Note: terminal fields return as objects with `confidence`, `origin`, and `value` rather than plain values. For more advanced response filtering, prefer the `&filter=` query parameter (see Step 4).

**Facet queries**

Reach for facet queries when the goal is aggregation or distribution analysis rather than retrieving individual entities — e.g. "what industries are most common among Berlin startups?" or "how are employees distributed across company sizes?". A facet query returns up to 1000 buckets ordered by frequency, each with a `count`, `value`, and `callbackQuery` you can use to drill into matching entities. They are not appropriate when the user wants to see individual entity records.

Key behaviors to know:

- Numeric and date fields are automatically grouped into ranges; custom ranges can be specified with `facet[a:b,b:c]:field`
- Date fields accept `day`, `week`, or `month` interval specifiers
- Results can be restricted to specific values with `facet["v1","v2"]:field`

See [Facet Queries](https://docs.diffbot.com/docs/facet-queries.md) for full syntax and examples.

**Subquery syntax for nested fields**

Use `{}` to co-constrain multiple conditions on the same nested object:

```
type:Organization locations.{city.name:"San Francisco" isCurrent:true}
```

This ensures San Francisco is a _current_ location, not that San Francisco is any location and some other location is current. Without `{}`, the two conditions are independent.

Not all fields support subqueries. Nestable fields have a composite type (e.g. `Location`, `Employment`) — check the ontology cache with `jq` to confirm. Attempting `{}` on a non-nested field returns: `Nested expression over non-nested list field [...] is not allowed`.

### Step 3 — Execute the request

```bash
curl -s "https://kg.diffbot.com/kg/v3/dql?token=${TOKEN}&query=ENCODED_QUERY&size=10"
```

Default `size=10`. Increase up to `size=50` if the user asks for more results. For pagination append `&from=10`, `&from=20`, etc.

### Step 4 — Format and display results

The response structure:

- `hits` — integer, total matching entity count
- `results` — integer, count of entities returned in this page
- `facet` — boolean, true when the query used `facet:`
- `data[]` — array of result objects:
  - Normal queries: each item has `score`, `entity` (the entity object), and `entity_ctx`
  - Facet queries: each item has `value`, `count`, and `callbackQuery` — there is no `entity` key and no separate `facets` top-level key

Always display the final DQL query in a plain text code block before the results, so the user can copy or iterate on it.

Pipe the curl output into an inline Python script to parse and display results. Write an appropriate formatter for the entity type being queried. Always use a heredoc to avoid shell quoting issues:

```bash
curl -s "https://kg.diffbot.com/kg/v3/dql?token=${TOKEN}&query=ENCODED_QUERY&size=10" | python3 << 'EOF'
import sys, json
data = json.load(sys.stdin)
hits = data['hits']  # plain integer
items = data['data']
# For normal queries: entity = item['entity']
# For facet queries: value = item['value'], count = item['count']
EOF
```

**Filtering the response**

Entity objects can be large. When analyzing results in context, use the `&filter=` query parameter to request only the fields needed — this minimizes data transfer and token usage. See [Filtering Fields](https://docs.diffbot.com/reference/filtering-fields.md) for full syntax.

```bash
curl -s "https://kg.diffbot.com/kg/v3/dql?token=${TOKEN}&query=ENCODED_QUERY&size=10&filter=name,nbEmployees,industries,locations"
```

When the user requests a full JSON dump or CSV download, omit the filter to get the complete response.

**Display format**

Render results as a terminal table (use plain ASCII borders). Choose columns appropriate to the entity type — for example, Organization: Name, Industry, Employees, Location, Website.

If the result set is large enough that the table would be unwieldy in the terminal (more than ~25 rows or more than ~6 columns), or if the user asks for file output, use the CSV export (see below) rather than writing a local file via Python.

**Exporting CSV/XLS**

Add `&format=csv` (or `xls`/`xlsx`) and `&exportspec=field,Name;field2,Name2` to export results directly without Python formatting. See [Exporting CSV](https://docs.diffbot.com/reference/exporting-csv.md).

After displaying results, offer to fetch the next page or refine the query.

