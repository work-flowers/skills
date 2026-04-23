# CRM Objects

Reading, creating, updating, searching, and deleting contacts, companies, deals, tickets, line items, products, and custom objects.

## Table of contents

- [Object types](#object-types)
- [Reading records](#reading-records)
- [Creating records](#creating-records)
- [Updating records](#updating-records)
- [Searching](#searching)
- [Deleting / archiving](#deleting--archiving)
- [Custom objects](#custom-objects)

## Object types

Standard objects and their `objectTypeId`:

| Object | Singular endpoint | `objectTypeId` |
|---|---|---|
| Contacts | `contacts` | `0-1` |
| Companies | `companies` | `0-2` |
| Deals | `deals` | `0-3` |
| Tickets | `tickets` | `0-5` |
| Products | `products` | `0-7` |
| Line items | `line_items` | `0-8` |
| Quotes | `quotes` | `0-14` |
| Calls | `calls` | `0-48` |
| Emails | `emails` | `0-49` |
| Meetings | `meetings` | `0-47` |
| Notes | `notes` | `0-46` |
| Tasks | `tasks` | `0-27` |

Custom objects have IDs like `2-12345678`. Get them via `client.crm.schemas.core_api.get_all()`.

## Reading records

### Single record by ID

```python
contact = client.crm.contacts.basic_api.get_by_id(
    contact_id="123",
    properties=["email", "firstname", "lastname", "lifecyclestage"],
    associations=["companies", "deals"],
)
print(contact.properties)
print(contact.associations)
```

### Single record by unique property (e.g. email)

```python
contact = client.crm.contacts.basic_api.get_by_id(
    contact_id="dennis@work.flowers",
    id_property="email",
    properties=["firstname", "lastname"],
)
```

### All records (paginated)

Use the `get_all_contacts` pattern in `core-patterns.md`. Substitute `companies`, `deals`, etc. for other object types. The SDK exposes `basic_api` for each.

### Reading associations

Associations on a record (via properties query):

```python
contact = client.crm.contacts.basic_api.get_by_id(
    contact_id="123",
    associations=["companies"],
)
for assoc in contact.associations.get("companies", {}).results or []:
    print(assoc.id, assoc.type)
```

For bulk association reads, use the Associations v4 API — see `associations.md`.

## Creating records

### Single contact

```python
from hubspot.crm.contacts import SimplePublicObjectInputForCreate

obj = SimplePublicObjectInputForCreate(
    properties={
        "email": "new@example.com",
        "firstname": "New",
        "lastname": "Contact",
        "lifecyclestage": "lead",
    }
)
created = client.crm.contacts.basic_api.create(
    simple_public_object_input_for_create=obj
)
print(f"Created contact {created.id}")
```

### Batch create

```python
from hubspot.crm.contacts import BatchInputSimplePublicObjectInputForCreate

def batch_create_contacts(client, inputs):
    """
    inputs: list of {"properties": {...}, "associations": [...]} dicts
    """
    all_results = []
    for i in range(0, len(inputs), 100):
        chunk = inputs[i:i + 100]
        req = BatchInputSimplePublicObjectInputForCreate(inputs=chunk)
        resp = client.crm.contacts.batch_api.create(
            batch_input_simple_public_object_input_for_create=req
        )
        all_results.extend(resp.results)
    return all_results
```

### Create with associations

Associate a new deal to an existing contact and company at creation time:

```python
deal_input = {
    "properties": {
        "dealname": "Terrascope — Annual renewal",
        "amount": "50000",
        "dealstage": "appointmentscheduled",
        "pipeline": "default",
    },
    "associations": [
        {
            "to": {"id": "CONTACT_ID"},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 3}],
        },
        {
            "to": {"id": "COMPANY_ID"},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 5}],
        },
    ],
}

from hubspot.crm.deals import BatchInputSimplePublicObjectInputForCreate
req = BatchInputSimplePublicObjectInputForCreate(inputs=[deal_input])
client.crm.deals.batch_api.create(
    batch_input_simple_public_object_input_for_create=req
)
```

Default `associationTypeId` values (HUBSPOT_DEFINED):

| From → To | Type ID |
|---|---|
| Contact → Company | 1 (primary), 279 |
| Contact → Deal | 4 |
| Deal → Contact | 3 |
| Deal → Company | 5 |
| Company → Contact | 2 (primary), 280 |

Full list via `/crm/v4/associations/{fromObjectType}/{toObjectType}/labels` — see `associations.md`.

## Updating records

### Single PATCH

```python
from hubspot.crm.contacts import SimplePublicObjectInput

obj = SimplePublicObjectInput(properties={"lifecyclestage": "customer"})
updated = client.crm.contacts.basic_api.update(
    contact_id="123",
    simple_public_object_input=obj,
)
```

### Batch update

See `core-patterns.md` → `batch_update_contacts`.

### Clearing a property

Set the value to an empty string:

```python
SimplePublicObjectInput(properties={"custom_field": ""})
```

## Searching

### Basic search

```python
from hubspot.crm.contacts import PublicObjectSearchRequest

req = PublicObjectSearchRequest(
    filter_groups=[{
        "filters": [
            {"propertyName": "email", "operator": "CONTAINS_TOKEN", "value": "@terrascope.earth"},
        ]
    }],
    properties=["email", "firstname", "lastname", "hs_lead_status"],
    limit=200,
)
page = client.crm.contacts.search_api.do_search(
    public_object_search_request=req
)
for r in page.results:
    print(r.properties)
```

### Filter operators

| Operator | Meaning |
|---|---|
| `EQ` | Equals |
| `NEQ` | Not equals |
| `LT`, `LTE`, `GT`, `GTE` | Less than / greater than (numeric, date) |
| `BETWEEN` | Requires `value` AND `highValue` |
| `IN` | Value in a list — use `values` (plural) array |
| `NOT_IN` | Value not in a list — `values` array |
| `HAS_PROPERTY` | Property has any value |
| `NOT_HAS_PROPERTY` | Property is empty/unset |
| `CONTAINS_TOKEN` | Substring match with wildcards |
| `NOT_CONTAINS_TOKEN` | Inverse |

### Combining filter groups

`filterGroups` are OR'd together. `filters` within a group are AND'd.
Max 5 filter groups, max 6 filters per group.

```python
# (email contains @terrascope.earth AND lifecyclestage = customer)
# OR
# (hs_lead_status = NEW)
filter_groups = [
    {
        "filters": [
            {"propertyName": "email", "operator": "CONTAINS_TOKEN", "value": "@terrascope.earth"},
            {"propertyName": "lifecyclestage", "operator": "EQ", "value": "customer"},
        ]
    },
    {
        "filters": [
            {"propertyName": "hs_lead_status", "operator": "EQ", "value": "NEW"},
        ]
    },
]
```

### Date filtering

Use epoch milliseconds:

```python
from datetime import datetime, timezone

cutoff_ms = int(datetime(2026, 1, 1, tzinfo=timezone.utc).timestamp() * 1000)
req = PublicObjectSearchRequest(
    filter_groups=[{
        "filters": [
            {"propertyName": "createdate", "operator": "GTE", "value": str(cutoff_ms)},
        ]
    }],
    sorts=[{"propertyName": "createdate", "direction": "DESCENDING"}],
    limit=200,
)
```

### Search gotchas

- **Eventually consistent**: freshly written records may take 5–30s to appear in search. For post-write reads, use `basic_api.get_by_id()`
- **Hard cap at 10,000 records** — see `core-patterns.md` → "Breaking the 10,000-record search cap"
- **Max 200 per page** on search; `basic_api.get_page()` is capped at 100
- **`CONTAINS_TOKEN` only works on string-type properties with tokenisation**, not enum values
- **Sorts are applied before the `after` cursor** — always include a stable sort (e.g. `hs_object_id`) for reliable pagination

## Deleting / archiving

Archive (soft delete, recoverable for 90 days):

```python
client.crm.contacts.basic_api.archive(contact_id="123")
```

Batch archive:

```python
from hubspot.crm.contacts import BatchInputSimplePublicObjectId

req = BatchInputSimplePublicObjectId(
    inputs=[{"id": "123"}, {"id": "456"}]
)
client.crm.contacts.batch_api.archive(
    batch_input_simple_public_object_id=req
)
```

There is no "hard delete" via API for most objects. GDPR deletion is a separate endpoint (`/crm/v3/objects/contacts/gdpr-delete`) — use only with explicit user instruction and document the action.

## Custom objects

Custom objects are addressed by their `objectType` (the internal name like `p12345_invoices`) or `objectTypeId` (e.g. `2-12345678`).

### List schemas

```python
schemas = client.crm.schemas.core_api.get_all()
for s in schemas.results:
    print(s.object_type_id, s.name, s.fully_qualified_name)
```

### Read a custom object record

Via the generic objects endpoint:

```python
record = client.crm.objects.basic_api.get_by_id(
    object_type="2-12345678",  # or the fully qualified name
    object_id="9876543",
    properties=["name", "custom_field_1"],
)
```

### Batch read / create / update custom objects

Same pattern as contacts, but use `client.crm.objects.batch_api.*` and pass `object_type` as the first argument:

```python
resp = client.crm.objects.batch_api.read(
    object_type="2-12345678",
    batch_read_input_simple_public_object_id=req,
)
```
