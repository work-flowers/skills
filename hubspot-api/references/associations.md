# Associations & Pipelines

Managing relationships between records (contacts ↔ companies, deals ↔ contacts, etc.) and configuring deal/ticket pipelines.

## Table of contents

- [Association concepts](#association-concepts)
- [Default association type IDs](#default-association-type-ids)
- [Reading associations](#reading-associations)
- [Creating associations](#creating-associations)
- [Deleting associations](#deleting-associations)
- [Association labels (custom types)](#association-labels-custom-types)
- [Pipelines and stages](#pipelines-and-stages)

## Association concepts

HubSpot has two categories of association types:

| Category | Description |
|---|---|
| `HUBSPOT_DEFINED` | Standard, system-managed association types (e.g. contact-to-company primary) |
| `USER_DEFINED` | Custom labels you create to annotate associations (e.g. "Decision Maker", "Technical Contact") |

**Every association has a direction** — contact → company is a different typeId from company → contact. Always check the direction.

There are also two API versions:

- **v3** — simple: `PUT /crm/v3/objects/{from}/{fromId}/associations/{to}/{toId}/{typeId}` — legacy, prefer v4
- **v4** — structured inputs with category + typeId, supports multiple labels per association

Use **v4** for all new work.

## Default association type IDs

Most common defaults (HUBSPOT_DEFINED). Direction matters — these are asymmetric.

### Contacts
| From → To | typeId | Notes |
|---|---|---|
| Contact → Company | 1 | Primary |
| Contact → Company | 279 | Unlabelled (non-primary) |
| Contact → Deal | 4 | |
| Contact → Ticket | 15 | |

### Companies
| From → To | typeId |
|---|---|
| Company → Contact | 2 (primary), 280 |
| Company → Deal | 6 (primary), 341 |
| Company → Ticket | 25 (primary), 339 |
| Company → Child company | 13 |
| Company → Parent company | 14 |

### Deals
| From → To | typeId |
|---|---|
| Deal → Contact | 3 |
| Deal → Company | 5 (primary), 341 |
| Deal → Line item | 19 |
| Deal → Ticket | 27 |
| Deal → Quote | 63 |

### Tickets
| From → To | typeId |
|---|---|
| Ticket → Contact | 16 |
| Ticket → Company | 26 (primary), 340 |
| Ticket → Deal | 28 |

To discover all available types between two object types programmatically:

```python
def list_association_types(client, from_type, to_type):
    resp = client.api_request({
        "path": f"/crm/v4/associations/{from_type}/{to_type}/labels",
        "method": "GET",
    })
    return resp.json().get("results", [])

for t in list_association_types(client, "deals", "contacts"):
    print(t)
# {"category": "HUBSPOT_DEFINED", "typeId": 3, "label": None}, ...
```

## Reading associations

### Single record: associations from contact to companies

```python
resp = client.api_request({
    "path": f"/crm/v4/objects/contacts/{contact_id}/associations/companies",
    "method": "GET",
    "qs": {"limit": 500},
})
associations = resp.json().get("results", [])
for a in associations:
    print(a["toObjectId"], a["associationTypes"])
# Each association has a list of types (one record can have multiple labels)
```

### Batch read associations

```python
resp = client.api_request({
    "path": "/crm/v4/associations/contacts/companies/batch/read",
    "method": "POST",
    "body": {
        "inputs": [{"id": str(cid)} for cid in contact_ids],
    },
})
# Response: {"results": [{"from": {"id": "..."}, "to": [{"toObjectId": "...", ...}]}]}
```

## Creating associations

### Default association (no label)

Use the v4 default endpoint — HubSpot sets the correct default type:

```python
resp = client.api_request({
    "path": f"/crm/v4/objects/contacts/{contact_id}/associations/default/companies/{company_id}",
    "method": "PUT",
})
```

### Specific association type

```python
resp = client.api_request({
    "path": f"/crm/v4/objects/deals/{deal_id}/associations/contacts/{contact_id}",
    "method": "PUT",
    "body": [
        {"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 3},
    ],
})
```

### Batch create with specific types

```python
resp = client.api_request({
    "path": "/crm/v4/associations/deals/contacts/batch/create",
    "method": "POST",
    "body": {
        "inputs": [
            {
                "from": {"id": "DEAL_ID_1"},
                "to": {"id": "CONTACT_ID_1"},
                "types": [
                    {"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 3},
                ],
            },
            {
                "from": {"id": "DEAL_ID_2"},
                "to": {"id": "CONTACT_ID_2"},
                "types": [
                    {"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 3},
                    {"associationCategory": "USER_DEFINED", "associationTypeId": 42},  # custom label
                ],
            },
        ],
    },
})
```

Max 100 per batch call.

## Deleting associations

### Remove all labels between two records

```python
client.api_request({
    "path": f"/crm/v4/objects/deals/{deal_id}/associations/contacts/{contact_id}",
    "method": "DELETE",
})
```

### Remove a specific label only

```python
client.api_request({
    "path": f"/crm/v4/objects/deals/{deal_id}/associations/contacts/{contact_id}/labels",
    "method": "DELETE",
    "body": [
        {"associationCategory": "USER_DEFINED", "associationTypeId": 42},
    ],
})
```

## Association labels (custom types)

Create custom labels to annotate associations (e.g. "Decision Maker", "Influencer", "Vendor"):

```python
resp = client.api_request({
    "path": "/crm/v4/associations/contacts/companies/labels",
    "method": "POST",
    "body": {
        "label": "Decision Maker",
        "name": "decision_maker",   # optional internal name
    },
})
# Response includes {"category": "USER_DEFINED", "typeId": <new_id>, ...}
```

List existing labels:

```python
resp = client.api_request({
    "path": "/crm/v4/associations/contacts/companies/labels",
    "method": "GET",
})
```

Delete a label:

```python
client.api_request({
    "path": f"/crm/v4/associations/contacts/companies/labels/{type_id}",
    "method": "DELETE",
})
```

**Note:** Deleting a label removes that annotation from all associations using it, but does not delete the underlying association (the default HUBSPOT_DEFINED type remains).

## Pipelines and stages

Pipelines apply to deals and tickets. Each object type can have multiple pipelines, and each pipeline has ordered stages.

### List pipelines

```python
# Deals
deal_pipelines = client.crm.pipelines.pipelines_api.get_all(object_type="deals")
for p in deal_pipelines.results:
    print(p.id, p.label, p.display_order)
    for stage in p.stages:
        print(f"  {stage.id} {stage.label} (prob: {stage.metadata.get('probability')})")

# Tickets — same pattern
ticket_pipelines = client.crm.pipelines.pipelines_api.get_all(object_type="tickets")
```

### Create a pipeline

```python
from hubspot.crm.pipelines import PipelineInput, PipelineStageInput

new = PipelineInput(
    label="Terrascope Renewal Pipeline",
    display_order=1,
    stages=[
        PipelineStageInput(
            label="Health Check",
            display_order=0,
            metadata={"probability": "0.1", "isClosed": "false"},
        ),
        PipelineStageInput(
            label="Renewal Discussion",
            display_order=1,
            metadata={"probability": "0.5", "isClosed": "false"},
        ),
        PipelineStageInput(
            label="Renewed",
            display_order=2,
            metadata={"probability": "1.0", "isClosed": "true"},
        ),
        PipelineStageInput(
            label="Churned",
            display_order=3,
            metadata={"probability": "0.0", "isClosed": "true"},
        ),
    ],
)
created = client.crm.pipelines.pipelines_api.create(
    object_type="deals",
    pipeline_input=new,
)
```

### Update a stage

```python
from hubspot.crm.pipelines import PipelineStagePatchInput

client.crm.pipelines.pipeline_stages_api.update(
    object_type="deals",
    pipeline_id=pipeline_id,
    stage_id=stage_id,
    pipeline_stage_patch_input=PipelineStagePatchInput(
        label="Updated stage label",
        metadata={"probability": "0.75"},
    ),
)
```

### Stage metadata reference

For **deals**:
- `probability`: decimal string between `"0.0"` and `"1.0"` (maps to "closed won" weighting)
- `isClosed`: `"true"` or `"false"`

For **tickets**:
- `ticketState`: `"OPEN"` or `"CLOSED"`

## Common patterns

### Find all companies associated with a contact

```python
resp = client.api_request({
    "path": f"/crm/v4/objects/contacts/{contact_id}/associations/companies",
    "method": "GET",
})
company_ids = [a["toObjectId"] for a in resp.json().get("results", [])]
```

### Set primary company for a contact

```python
# The primary association uses a specific typeId (HUBSPOT_DEFINED 1 for contact→company)
client.api_request({
    "path": f"/crm/v4/objects/contacts/{contact_id}/associations/companies/{company_id}",
    "method": "PUT",
    "body": [
        {"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 1},
    ],
})
```

### Audit: find all deals missing a company association

```python
from hubspot.crm.deals import PublicObjectSearchRequest

req = PublicObjectSearchRequest(
    filter_groups=[{
        "filters": [
            {"propertyName": "associations.company", "operator": "NOT_HAS_PROPERTY"},
        ]
    }],
    properties=["dealname", "amount"],
    limit=200,
)
# Note: the NOT_HAS_PROPERTY approach on associations is limited.
# Safer: fetch all deals, then batch-read associations, then diff.
```
