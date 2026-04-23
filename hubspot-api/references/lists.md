# Lists (Segments)

The v3 Lists API. Lists are now called "segments" in the UI but the API endpoints still use `lists`. Supports contacts, companies, deals, tickets, and custom objects.

## Table of contents

- [List types](#list-types)
- [Creating a list](#creating-a-list)
- [Fetching lists](#fetching-lists)
- [Managing memberships (static/manual lists)](#managing-memberships-staticmanual-lists)
- [Filter branch syntax (dynamic lists)](#filter-branch-syntax-dynamic-lists)
- [Updating a list's filter](#updating-a-lists-filter)
- [Deleting and restoring](#deleting-and-restoring)

**Important versioning note:** the v1 Contact Lists API sunsets on **30 April 2026**. All new work must use v3 (`/crm/v3/lists/...`). The SDK exposes this under `client.crm.lists`.

## List types

Three processing types. Choose carefully — you can't convert between them freely.

| Type | Behaviour |
|---|---|
| `MANUAL` | Static list. Records are added/removed explicitly via API or UI; nothing happens automatically. |
| `DYNAMIC` | Active list. Filters re-evaluate automatically; memberships update as records change. |
| `SNAPSHOT` | Records are frozen at creation time based on a filter — doesn't update. |

Active (dynamic) lists can be converted to static via `/crm/v3/lists/{listId}/convert-to-static`. The reverse is not possible.

## Creating a list

### Static (manual) list

```python
resp = client.api_request({
    "path": "/crm/v3/lists",
    "method": "POST",
    "body": {
        "name": "Q2 2026 Target Accounts",
        "objectTypeId": "0-1",              # 0-1 = contact, 0-2 = company, 0-3 = deal
        "processingType": "MANUAL",
    },
})
list_data = resp.json()["list"]
list_id = list_data["listId"]
print(f"Created list {list_id}")
```

### Dynamic list (with filters)

```python
resp = client.api_request({
    "path": "/crm/v3/lists",
    "method": "POST",
    "body": {
        "name": "Active Terrascope Contacts",
        "objectTypeId": "0-1",
        "processingType": "DYNAMIC",
        "filterBranch": {
            "filterBranchType": "AND",
            "filterBranches": [],
            "filters": [
                {
                    "filterType": "PROPERTY",
                    "property": "email",
                    "operation": {
                        "operationType": "ALL_PROPERTY",
                        "operator": "IS_KNOWN",
                    },
                },
                {
                    "filterType": "PROPERTY",
                    "property": "company",
                    "operation": {
                        "operationType": "STRING",
                        "operator": "IS_EQUAL_TO",
                        "value": "Terrascope",
                    },
                },
            ],
        },
    },
})
```

## Fetching lists

### By ID

```python
resp = client.api_request({
    "path": "/crm/v3/lists/612",
    "method": "GET",
    "qs": {"includeFilters": "true"},
})
```

### By name + object type

```python
resp = client.api_request({
    "path": "/crm/v3/lists/object-type-id/0-1/name/My%20List",
    "method": "GET",
})
```

### Search lists

```python
resp = client.api_request({
    "path": "/crm/v3/lists/search",
    "method": "POST",
    "body": {
        "query": "Terrascope",
        "processingTypes": ["DYNAMIC", "MANUAL"],
        "objectTypeId": "0-1",
        "offset": 0,
        "count": 50,
    },
})
```

## Managing memberships (static/manual lists)

### Add records to a list

```python
# PUT /crm/v3/lists/{listId}/memberships/add
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/memberships/add",
    "method": "PUT",
    "body": [contact_id_1, contact_id_2, contact_id_3],  # plain list of IDs
})
```

### Remove records

```python
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/memberships/remove",
    "method": "PUT",
    "body": [contact_id_1, contact_id_2],
})
```

### Add all records from another list

```python
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/memberships/add-from/{source_list_id}",
    "method": "PUT",
})
```

### Fetch memberships (with pagination)

```python
def get_list_members(client, list_id):
    members = []
    after = None
    while True:
        qs = {"limit": 250}
        if after:
            qs["after"] = after
        resp = client.api_request({
            "path": f"/crm/v3/lists/{list_id}/memberships",
            "method": "GET",
            "qs": qs,
        })
        data = resp.json()
        members.extend(data.get("results", []))
        paging = data.get("paging", {}).get("next", {})
        if not paging:
            break
        after = paging.get("after")
    return members
```

### Memberships with join order (when they joined the list)

```python
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/memberships/join-order",
    "method": "GET",
    "qs": {"limit": 250},
})
```

## Filter branch syntax (dynamic lists)

The `filterBranch` structure is recursive: a branch has a type (`AND` or `OR`), an array of child branches, and an array of filters.

```
filterBranch
├── filterBranchType: "AND" | "OR"
├── filterBranches: [ ...nested branches... ]
└── filters: [ ...leaf filters... ]
```

### Filter operations by data type

**String (text properties):**

```python
{
    "filterType": "PROPERTY",
    "property": "email",
    "operation": {
        "operationType": "STRING",
        "operator": "IS_EQUAL_TO" | "IS_NOT_EQUAL_TO" | "CONTAINS" | "DOES_NOT_CONTAIN"
                  | "STARTS_WITH" | "ENDS_WITH",
        "value": "...",
    },
}
```

**Number:**

```python
{
    "filterType": "PROPERTY",
    "property": "amount",
    "operation": {
        "operationType": "NUMBER",
        "operator": "IS_EQUAL_TO" | "IS_GREATER_THAN" | "IS_GREATER_THAN_OR_EQUAL_TO"
                  | "IS_LESS_THAN" | "IS_LESS_THAN_OR_EQUAL_TO" | "IS_BETWEEN",
        "value": 10000,
        "highValue": 50000,  # only for IS_BETWEEN
    },
}
```

**Enumeration (dropdown, checkbox, etc.):**

```python
{
    "filterType": "PROPERTY",
    "property": "lifecyclestage",
    "operation": {
        "operationType": "ENUMERATION",
        "operator": "IS_ANY_OF" | "IS_NONE_OF",
        "values": ["lead", "marketingqualifiedlead"],
    },
}
```

**Property has/doesn't have a value (any type):**

```python
{
    "filterType": "PROPERTY",
    "property": "email",
    "operation": {
        "operationType": "ALL_PROPERTY",
        "operator": "IS_KNOWN" | "IS_NOT_KNOWN",
        "includeObjectsWithNoValueSet": False,  # for IS_NOT_KNOWN
    },
}
```

**Date:**

```python
{
    "filterType": "PROPERTY",
    "property": "createdate",
    "operation": {
        "operationType": "DATE",
        "operator": "IS_BEFORE" | "IS_AFTER" | "IS_BETWEEN" | "IS_EQUAL_TO"
                  | "IS_IN_THE_LAST" | "IS_IN_THE_NEXT",
        "value": "2026-01-01",
        "timezoneSource": "CUSTOMER",
    },
}
```

### Email subscription filter

```python
{
    "filterType": "EMAIL_SUBSCRIPTION",
    "acceptedStatuses": ["OPT_IN"],        # or "OPT_OUT", "NEVER_SUBSCRIBED"
    "subscriptionIds": ["81537745"],
}
```

### In-list filter (membership of another list)

```python
{
    "filterType": "IN_LIST",
    "operator": "IN_LIST" | "NOT_IN_LIST",
    "listId": 42,
}
```

### Complex example: (A AND B) OR (C)

```python
"filterBranch": {
    "filterBranchType": "OR",
    "filterBranches": [
        {
            "filterBranchType": "AND",
            "filterBranches": [],
            "filters": [
                {  # A: amount >= 10000
                    "filterType": "PROPERTY",
                    "property": "amount",
                    "operation": {"operationType": "NUMBER", "operator": "IS_GREATER_THAN_OR_EQUAL_TO", "value": 10000},
                },
                {  # B: deal_type = standard
                    "filterType": "PROPERTY",
                    "property": "dealtype",
                    "operation": {"operationType": "ENUMERATION", "operator": "IS_ANY_OF", "values": ["standard"]},
                },
            ],
        },
        {
            "filterBranchType": "AND",
            "filterBranches": [],
            "filters": [
                {  # C: deal_type = hot
                    "filterType": "PROPERTY",
                    "property": "dealtype",
                    "operation": {"operationType": "ENUMERATION", "operator": "IS_ANY_OF", "values": ["hot"]},
                },
            ],
        },
    ],
    "filters": [],
}
```

## Updating a list's filter

PUT the new filter branch. The list will re-process memberships.

```python
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/update-list-filters",
    "method": "PUT",
    "qs": {"enrollObjectsInWorkflows": "false"},
    "body": {"filterBranch": new_filter_branch},
})
```

The `enrollObjectsInWorkflows` flag controls whether records newly added by the filter change are enrolled in workflows that trigger on list membership. Default to `false` unless you want that side-effect.

## Updating a list's name

```python
resp = client.api_request({
    "path": f"/crm/v3/lists/{list_id}/update-list-name",
    "method": "PUT",
    "qs": {"listName": "New Name"},
})
```

## Deleting and restoring

Delete (soft — recoverable for 90 days):

```python
client.api_request({
    "path": f"/crm/v3/lists/{list_id}",
    "method": "DELETE",
})
```

Restore:

```python
client.api_request({
    "path": f"/crm/v3/lists/{list_id}/restore",
    "method": "PUT",
})
```

## Known limitations

- **Date-related filters cannot always be created via API** in all edge cases — some date operators (particularly rolling windows) are restricted. If you get a 400 on a list create with date filters, try creating the list in the UI and inspecting the filter branch via GET to see the exact structure expected.
- **Static list manual adds are NOT evaluated against other dynamic lists' filters** — adding a contact to static list A doesn't affect dynamic list B even if A's members match B's filters, unless workflows are configured to link them.
