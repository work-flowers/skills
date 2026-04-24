---
name: notion-python-api
description: Write and execute Python scripts that call the Notion API directly using the `notion-client` SDK. Use this skill for complex Notion operations that exceed the MCP connector's practical limits — specifically large database queries requiring pagination, cross-database relation traversal, bulk create/update operations, aggregations and counts, and data exports to CSV or markdown. Trigger whenever the user asks to query, filter, aggregate, count, summarise, export, cross-reference, or bulk-modify data across Notion databases — whether in Dennis's own workspace or a client's workspace. Also trigger when the task would require more than ~5 sequential MCP calls to complete, or when the user wants to interact with a Notion database programmatically. Do NOT trigger for simple single-page reads or writes — the Notion MCP connector handles those fine.
---

# Notion Python API — Direct SDK Access

## When to use this skill

Use this skill instead of the Notion MCP connector when:

- **Large queries** — paginating through dozens or hundreds of database results
- **Cross-database joins** — following relation properties across databases (e.g. company → meeting notes → tasks)
- **Bulk operations** — creating or updating many pages in a batch
- **Aggregations** — counting, grouping, or summarising database records
- **Data exports** — pulling database contents to CSV, markdown, or structured output
- **Complex filters** — combining multiple filter conditions that are awkward via MCP
- **Client workspaces** — querying databases in a client's Notion workspace (requires their integration token)

For simple reads/writes to a single page, use the Notion MCP connector directly.

## Workflow

1. **Install the SDK** (once per session):
   ```bash
   pip install notion-client --break-system-packages
   ```

2. **Check authentication**:
   ```python
   import os
   token = os.environ.get("NOTION_TOKEN")
   if not token:
       raise SystemExit("NOTION_TOKEN environment variable is not set. Please configure it before proceeding.")
   ```

3. **Identify the target databases** — either from the workspace schema reference (for Dennis's CRM) or by asking the user for database IDs / URLs.

4. **Write a purpose-built script** tailored to the specific query — do not make dozens of sequential MCP calls.

5. **Execute, process, and present** the results (or save to file).

## Authentication

- Always read from `os.environ["NOTION_TOKEN"]` — never hardcode tokens
- The token is a Notion internal integration token (starts with `ntn_`)
- If the env var is missing, tell the user and stop — do not prompt for inline input
- **Client workspaces**: If querying a client's workspace, the user may need to provide a different token. Ask which workspace the query targets and whether the current token has access.

## Reference files

| File | When to read |
|---|---|
| `references/workspace-schema.md` | When querying Dennis's own CRM databases — contains database IDs, relation maps, and property names for the work.flowers Notion workspace |
| `references/query-patterns.md` | When writing any non-trivial script — contains tested, copy-paste-ready patterns for pagination, filters, relation traversal, property extraction, CSV export, and block reading |

**For Dennis's CRM queries**, always read `references/workspace-schema.md` first — it has the database IDs and relation property names you'll need.

**For client or unknown databases**, you may need to:
- Ask the user for the database ID (from the Notion URL or share link)
- Use `notion.databases.retrieve(database_id=...)` to inspect the schema and discover property names and types
- Build the query dynamically based on the discovered schema

### Discovering an unknown database schema

```python
# Extract database_id from a Notion URL:
# https://www.notion.so/workspace/abc123def456...?v=...
# The 32-character hex string (with hyphens inserted) is the database_id

schema = notion.databases.retrieve(database_id="your-database-id")
print(f"Title: {schema['title'][0]['plain_text']}")
print(f"\nProperties:")
for name, prop in schema["properties"].items():
    print(f"  {name}: {prop['type']}")
```

## Key constraints

- Always use `notion-client` Python SDK (v3.0.0+), not raw `requests`
- **Always paginate** — never assume a single page of results is complete
- Add `time.sleep(0.35)` between batch API calls (Notion rate limit: 3 req/s)
- Include error handling for rate limits (HTTP 429), missing properties, and invalid IDs
- All scripts must be self-contained — no external config files beyond the env var
- When extracting properties, use the helper patterns from `query-patterns.md` to handle Notion's verbose property format

## Common tasks

| User request | Approach |
|---|---|
| "How many [records] by [status/category]?" | Query target DB → group by property → count |
| "Get all [related records] for [entity]" | Find parent page → read relation property → fetch linked pages |
| "Export [database] to CSV" | Paginated query → flatten properties → write CSV |
| "Which records have [relation A] but not [relation B]?" | Cross-database query with relation filters |
| "Create pages from this list" | Batch create with rate limiting |
| "What's in this database?" | Retrieve schema → inspect properties → query |

## Relationship to other skills

- **`notion-crm-relations`** — teaches relation traversal strategy via MCP for Dennis's CRM. This skill uses Python for bulk/complex operations and works with any Notion workspace.
- **`notion-knowledge-capture`** / **`notion-meeting-intelligence`** — use MCP for writes. This skill complements them for bulk/complex reads.