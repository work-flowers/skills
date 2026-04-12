# Notion Python SDK — Reusable Query Patterns

All patterns use `notion-client` v3.0.0+. Import boilerplate:

```python
import os
import time
import csv
import json
from notion_client import Client

notion = Client(auth=os.environ["NOTION_TOKEN"])
```

---

## Table of Contents

1. [Paginated database query](#1-paginated-database-query)
2. [Filtered database query](#2-filtered-database-query)
3. [Relation traversal](#3-relation-traversal)
4. [Bulk page creation](#4-bulk-page-creation)
5. [Property extraction helpers](#5-property-extraction-helpers)
6. [Export to CSV](#6-export-to-csv)
7. [Block content reading](#7-block-content-reading)
8. [Error handling](#8-error-handling)

---

## 1. Paginated database query

Always use this pattern — never assume a single response contains all results.

```python
def query_all(database_id: str, filter_obj: dict | None = None, sorts: list | None = None) -> list:
    """Query a Notion database with automatic pagination. Returns all matching pages."""
    results = []
    has_more = True
    next_cursor = None

    while has_more:
        kwargs = {"database_id": database_id}
        if filter_obj:
            kwargs["filter"] = filter_obj
        if sorts:
            kwargs["sorts"] = sorts
        if next_cursor:
            kwargs["start_cursor"] = next_cursor

        response = notion.databases.query(**kwargs)
        results.extend(response["results"])
        has_more = response.get("has_more", False)
        next_cursor = response.get("next_cursor")

        if has_more:
            time.sleep(0.35)  # Rate limit: 3 req/s

    return results
```

---

## 2. Filtered database query

Notion filters use a specific JSON structure. Here are the common patterns:

### Status / Select filter

```python
# Single status match
filter_obj = {
    "property": "Status",
    "status": {"equals": "Active"}
}

# Select property
filter_obj = {
    "property": "Type",
    "select": {"equals": "Retainer"}
}
```

### Multi-select filter

```python
filter_obj = {
    "property": "Tags",
    "multi_select": {"contains": "Automation"}
}
```

### Text / Title filter

```python
# Title (for the Name / title property)
filter_obj = {
    "property": "Name",
    "title": {"equals": "Acme Corp"}
}

# Rich text
filter_obj = {
    "property": "Description",
    "rich_text": {"contains": "zapier"}
}
```

### Date filter

```python
from datetime import datetime, timedelta

# After a specific date
filter_obj = {
    "property": "Date",
    "date": {"after": "2026-01-01"}
}

# In the past 30 days
thirty_days_ago = (datetime.now() - timedelta(days=30)).strftime("%Y-%m-%d")
filter_obj = {
    "property": "Date",
    "date": {"on_or_after": thirty_days_ago}
}

# Date is not empty
filter_obj = {
    "property": "Date",
    "date": {"is_not_empty": True}
}
```

### Relation filter

```python
# Has a specific related page
filter_obj = {
    "property": "Company",
    "relation": {"contains": "page-id-here"}
}

# Relation is not empty (has at least one link)
filter_obj = {
    "property": "Deals",
    "relation": {"is_not_empty": True}
}

# Relation is empty
filter_obj = {
    "property": "Meeting Notes",
    "relation": {"is_empty": True}
}
```

### Number filter

```python
filter_obj = {
    "property": "Amount",
    "number": {"greater_than": 10000}
}
```

### Checkbox filter

```python
filter_obj = {
    "property": "Archived",
    "checkbox": {"equals": False}
}
```

### Compound filters (AND / OR)

```python
# AND — all conditions must match
filter_obj = {
    "and": [
        {"property": "Status", "status": {"equals": "Active"}},
        {"property": "Amount", "number": {"greater_than": 5000}},
    ]
}

# OR — any condition matches
filter_obj = {
    "or": [
        {"property": "Status", "status": {"equals": "Active"}},
        {"property": "Status", "status": {"equals": "In Progress"}},
    ]
}

# Nested: (Status = Active OR In Progress) AND Amount > 5000
filter_obj = {
    "and": [
        {
            "or": [
                {"property": "Status", "status": {"equals": "Active"}},
                {"property": "Status", "status": {"equals": "In Progress"}},
            ]
        },
        {"property": "Amount", "number": {"greater_than": 5000}},
    ]
}
```

### Sorts

```python
sorts = [
    {"property": "Date", "direction": "descending"},
    {"property": "Name", "direction": "ascending"},
]
```

---

## 3. Relation traversal

### Find a record by title, then fetch related records

```python
def find_record_by_title(database_id: str, title_property: str, name: str) -> dict | None:
    """Find a page by exact title match in any database."""
    results = notion.databases.query(
        database_id=database_id,
        filter={"property": title_property, "title": {"equals": name}},
    )["results"]
    return results[0] if results else None


def get_relation_page_ids(page: dict, property_name: str) -> list[str]:
    """Extract page IDs from a relation property."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "relation":
        return [rel["id"] for rel in prop["relation"]]
    return []


def fetch_pages(page_ids: list[str]) -> list[dict]:
    """Fetch multiple pages by ID with rate limiting."""
    pages = []
    for pid in page_ids:
        try:
            page = notion.pages.retrieve(page_id=pid)
            pages.append(page)
        except Exception as e:
            print(f"Warning: Could not fetch page {pid}: {e}")
        time.sleep(0.35)
    return pages
```

### Full example: find a parent record and fetch its related pages

```python
# Example: find a company and get its meeting notes
# (adapt database_id and property names to your workspace)
parent = find_record_by_title("your-database-id", "Name", "Acme Corp")
if not parent:
    print("Record not found")
else:
    related_ids = get_relation_page_ids(parent, "Meeting Notes")
    print(f"Found {len(related_ids)} related pages")
    related_pages = fetch_pages(related_ids)
    for p in related_pages:
        title = get_title(p)
        date = get_date(p, "Date")
        print(f"  - {date}: {title}")
```

### Alternative: query from the other side using a relation filter

When the child database has a relation back to the parent, you can query directly — this is faster for large relation sets:

```python
# Find all child records related to a specific parent page
parent_page_id = parent["id"]
children = query_all(
    "child-database-id",
    filter_obj={"property": "Parent Relation", "relation": {"contains": parent_page_id}},
    sorts=[{"property": "Date", "direction": "descending"}],
)
```

This avoids fetching each page individually.

---

## 4. Bulk page creation

```python
def bulk_create_pages(database_id: str, pages_data: list[dict], delay: float = 0.35) -> list[dict]:
    """
    Create multiple pages with rate limiting.
    Each item in pages_data should be a dict of {property_name: value} in Notion's property format.
    """
    created = []
    for i, props in enumerate(pages_data):
        try:
            page = notion.pages.create(
                parent={"database_id": database_id},
                properties=props,
            )
            created.append(page)
            print(f"  Created {i + 1}/{len(pages_data)}: {page['id']}")
        except Exception as e:
            print(f"  Error creating page {i + 1}: {e}")
        time.sleep(delay)
    return created
```

### Property format for page creation

```python
# Title property
{"Name": {"title": [{"text": {"content": "My Page Title"}}]}}

# Rich text
{"Description": {"rich_text": [{"text": {"content": "Some text"}}]}}

# Select
{"Status": {"select": {"name": "Active"}}}

# Multi-select
{"Tags": {"multi_select": [{"name": "Tag1"}, {"name": "Tag2"}]}}

# Number
{"Amount": {"number": 15000}}

# Date
{"Due Date": {"date": {"start": "2026-04-15"}}}

# Checkbox
{"Done": {"checkbox": True}}

# Relation
{"Company": {"relation": [{"id": "page-id-here"}]}}

# URL
{"Website": {"url": "https://example.com"}}

# Email
{"Email": {"email": "hello@example.com"}}
```

---

## 5. Property extraction helpers

Notion's property format is deeply nested. These helpers extract clean values.

```python
def get_title(page: dict, property_name: str = "Name") -> str:
    """Extract title text from a page."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "title":
        parts = prop.get("title", [])
        return "".join(p.get("plain_text", "") for p in parts)
    return ""


def get_rich_text(page: dict, property_name: str) -> str:
    """Extract rich text as plain string."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "rich_text":
        parts = prop.get("rich_text", [])
        return "".join(p.get("plain_text", "") for p in parts)
    return ""


def get_select(page: dict, property_name: str) -> str | None:
    """Extract select value name."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "select" and prop.get("select"):
        return prop["select"]["name"]
    return None


def get_status(page: dict, property_name: str) -> str | None:
    """Extract status value name."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "status" and prop.get("status"):
        return prop["status"]["name"]
    return None


def get_multi_select(page: dict, property_name: str) -> list[str]:
    """Extract multi-select values as list of names."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "multi_select":
        return [item["name"] for item in prop.get("multi_select", [])]
    return []


def get_date(page: dict, property_name: str) -> str | None:
    """Extract date start value (ISO string)."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "date" and prop.get("date"):
        return prop["date"].get("start")
    return None


def get_number(page: dict, property_name: str) -> float | None:
    """Extract number value."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "number":
        return prop.get("number")
    return None


def get_relation_ids(page: dict, property_name: str) -> list[str]:
    """Extract page IDs from a relation property."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "relation":
        return [rel["id"] for rel in prop.get("relation", [])]
    return []


def get_rollup(page: dict, property_name: str):
    """Extract rollup value — returns the inner value depending on rollup type."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") != "rollup":
        return None
    rollup = prop.get("rollup", {})
    rtype = rollup.get("type")
    if rtype == "number":
        return rollup.get("number")
    elif rtype == "date":
        return rollup.get("date", {}).get("start") if rollup.get("date") else None
    elif rtype == "array":
        return rollup.get("array", [])
    return None


def get_formula(page: dict, property_name: str):
    """Extract formula result value."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") != "formula":
        return None
    formula = prop.get("formula", {})
    ftype = formula.get("type")
    return formula.get(ftype)


def get_url(page: dict, property_name: str) -> str | None:
    """Extract URL value."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "url":
        return prop.get("url")
    return None


def get_email(page: dict, property_name: str) -> str | None:
    """Extract email value."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "email":
        return prop.get("email")
    return None


def get_checkbox(page: dict, property_name: str) -> bool:
    """Extract checkbox value."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "checkbox":
        return prop.get("checkbox", False)
    return False


def get_people(page: dict, property_name: str) -> list[str]:
    """Extract people names from a people property."""
    prop = page["properties"].get(property_name, {})
    if prop.get("type") == "people":
        return [p.get("name", p.get("id", "")) for p in prop.get("people", [])]
    return []
```

---

## 6. Export to CSV

```python
def export_database_to_csv(
    database_id: str,
    output_path: str,
    property_extractors: dict[str, callable],
    filter_obj: dict | None = None,
    sorts: list | None = None,
) -> int:
    """
    Query a database and export to CSV.

    property_extractors: dict mapping column names to functions that take a page
    and return the value. Example:
        {
            "Name": lambda p: get_title(p),
            "Status": lambda p: get_status(p, "Status"),
            "Amount": lambda p: get_number(p, "Amount"),
        }

    Returns the number of rows written.
    """
    pages = query_all(database_id, filter_obj=filter_obj, sorts=sorts)

    if not pages:
        print("No results found.")
        return 0

    with open(output_path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=list(property_extractors.keys()))
        writer.writeheader()
        for page in pages:
            row = {}
            for col_name, extractor in property_extractors.items():
                try:
                    val = extractor(page)
                    # Convert lists to semicolon-separated strings for CSV
                    if isinstance(val, list):
                        val = "; ".join(str(v) for v in val)
                    row[col_name] = val
                except Exception:
                    row[col_name] = ""
            writer.writerow(row)

    print(f"Exported {len(pages)} rows to {output_path}")
    return len(pages)
```

### Example: export a database to CSV

```python
export_database_to_csv(
    database_id="your-database-id",
    output_path="/home/claude/export.csv",
    property_extractors={
        "Name": lambda p: get_title(p),
        "Status": lambda p: get_status(p, "Status"),
        "Website": lambda p: get_url(p, "Website"),
        "Related Count": lambda p: len(get_relation_ids(p, "Related Items")),
    },
)
```

---

## 7. Block content reading

Read all blocks from a page as markdown.

```python
def read_page_blocks_as_markdown(page_id: str, indent: int = 0) -> str:
    """Recursively read all blocks from a page and convert to markdown."""
    lines = []
    has_more = True
    next_cursor = None
    prefix = "  " * indent

    while has_more:
        kwargs = {"block_id": page_id}
        if next_cursor:
            kwargs["start_cursor"] = next_cursor

        response = notion.blocks.children.list(**kwargs)
        blocks = response.get("results", [])
        has_more = response.get("has_more", False)
        next_cursor = response.get("next_cursor")

        for block in blocks:
            btype = block.get("type", "")
            content = block.get(btype, {})

            text = _extract_rich_text(content.get("rich_text", []))

            if btype == "paragraph":
                lines.append(f"{prefix}{text}")
            elif btype.startswith("heading_"):
                level = int(btype[-1])
                lines.append(f"{prefix}{'#' * level} {text}")
            elif btype == "bulleted_list_item":
                lines.append(f"{prefix}- {text}")
            elif btype == "numbered_list_item":
                lines.append(f"{prefix}1. {text}")
            elif btype == "to_do":
                checked = "x" if content.get("checked") else " "
                lines.append(f"{prefix}- [{checked}] {text}")
            elif btype == "toggle":
                lines.append(f"{prefix}<details><summary>{text}</summary>")
            elif btype == "code":
                lang = content.get("language", "")
                lines.append(f"{prefix}```{lang}")
                lines.append(f"{prefix}{text}")
                lines.append(f"{prefix}```")
            elif btype == "quote":
                lines.append(f"{prefix}> {text}")
            elif btype == "callout":
                emoji = content.get("icon", {}).get("emoji", "")
                lines.append(f"{prefix}> {emoji} {text}")
            elif btype == "divider":
                lines.append(f"{prefix}---")
            elif btype == "table_of_contents":
                lines.append(f"{prefix}[Table of Contents]")
            elif btype == "bookmark":
                url = content.get("url", "")
                lines.append(f"{prefix}[Bookmark]({url})")
            elif btype == "image":
                url = content.get("file", {}).get("url", content.get("external", {}).get("url", ""))
                lines.append(f"{prefix}![image]({url})")
            else:
                if text:
                    lines.append(f"{prefix}{text}")

            # Recurse into children
            if block.get("has_children"):
                child_md = read_page_blocks_as_markdown(block["id"], indent=indent + 1)
                lines.append(child_md)

        if has_more:
            time.sleep(0.35)

    return "\n".join(lines)


def _extract_rich_text(rich_text_array: list) -> str:
    """Convert Notion rich_text array to plain text with basic markdown formatting."""
    parts = []
    for rt in rich_text_array:
        text = rt.get("plain_text", "")
        annotations = rt.get("annotations", {})
        if annotations.get("bold"):
            text = f"**{text}**"
        if annotations.get("italic"):
            text = f"*{text}*"
        if annotations.get("strikethrough"):
            text = f"~~{text}~~"
        if annotations.get("code"):
            text = f"`{text}`"
        href = rt.get("href")
        if href:
            text = f"[{text}]({href})"
        parts.append(text)
    return "".join(parts)
```

---

## 8. Error handling

### Rate limit retry

```python
from notion_client import APIResponseError

def safe_query(database_id: str, **kwargs) -> dict:
    """Query with automatic retry on rate limit."""
    max_retries = 3
    for attempt in range(max_retries):
        try:
            return notion.databases.query(database_id=database_id, **kwargs)
        except APIResponseError as e:
            if e.status == 429 and attempt < max_retries - 1:
                wait = 2 ** attempt  # Exponential backoff: 1s, 2s, 4s
                print(f"Rate limited. Waiting {wait}s...")
                time.sleep(wait)
            else:
                raise
```

### Missing property guard

```python
def safe_get_property(page: dict, property_name: str) -> dict:
    """Safely get a property, returning empty dict if missing."""
    return page.get("properties", {}).get(property_name, {})
```

### Validate environment on startup

```python
def init_notion_client() -> Client:
    """Initialise client with validation."""
    token = os.environ.get("NOTION_TOKEN")
    if not token:
        raise SystemExit(
            "NOTION_TOKEN environment variable is not set.\n"
            "Please set it before running this script."
        )
    if not token.startswith("ntn_"):
        print("Warning: Token does not start with 'ntn_' — ensure it is a valid internal integration token.")
    return Client(auth=token)
```
