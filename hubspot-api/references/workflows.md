# Workflows (Automation v4 / Flows)

The v4 Automation API manages workflows (internally called "flows") across all object types. The legacy v3 Workflows API only handles contact-based workflows and is limited.

**Status:** v4 is out of beta for basic operations but some edge cases around update (PUT) remain finicky. Always check the changelog before running in production.

## Table of contents

- [Required scope](#required-scope)
- [Listing workflows](#listing-workflows)
- [Reading a workflow](#reading-a-workflow)
- [Updating a workflow](#updating-a-workflow)
- [Creating a workflow](#creating-a-workflow)
- [Deleting / disabling](#deleting--disabling)
- [Enrollment and performance data](#enrollment-and-performance-data)
- [Mapping v3 workflow IDs to v4 flow IDs](#mapping-v3-workflow-ids-to-v4-flow-ids)
- [Custom-coded workflow actions](#custom-coded-workflow-actions)

## Required scope

Your private app needs the `automation` scope. For workflows that reference sensitive data, also add the relevant sensitive-data scopes (e.g. `crm.objects.contacts.sensitive.read.v2`).

## Listing workflows

```python
def list_all_flows(client):
    flows = []
    after = None
    while True:
        qs = {"limit": 100}
        if after:
            qs["after"] = after
        resp = client.api_request({
            "path": "/automation/v4/flows",
            "method": "GET",
            "qs": qs,
        })
        data = resp.json()
        flows.extend(data.get("results", []))
        paging = data.get("paging", {}).get("next", {})
        if not paging:
            break
        after = paging.get("after")
    return flows
```

Response includes only summary fields: `id`, `isEnabled`, `objectTypeId`, `name`, and the latest `revisionId`. For the full definition, fetch by ID.

## Reading a workflow

### By ID (full definition)

```python
resp = client.api_request({
    "path": f"/automation/v4/flows/{flow_id}",
    "method": "GET",
})
flow = resp.json()
```

The response contains the entire flow: enrollment criteria, actions, branches, revision metadata. Structure:

```
{
  "id": "12345",
  "type": "CONTACT_FLOW",           # or COMPANY_FLOW, DEAL_FLOW, ...
  "revisionId": "7",
  "isEnabled": true,
  "name": "...",
  "enrollmentCriteria": { ... },
  "actions": [ ... ],
  "goal": { ... },
  "suppressionListIds": [],
  "timeWindows": [],
  "blockedDates": [],
  ...
}
```

### Batch read

```python
resp = client.api_request({
    "path": "/automation/v4/flows/batch/read",
    "method": "POST",
    "body": {
        "inputs": [
            {"flowId": "12345", "type": "FLOW_ID"},
            {"flowId": "67890", "type": "FLOW_ID"},
        ],
    },
})
```

## Updating a workflow

**⚠️ DESTRUCTIVE:** a PUT replaces the entire flow. Fields omitted from the body are removed — including actions. Always follow the fetch-mutate-put pattern.

### The safe pattern

```python
# 1. Fetch current state
resp = client.api_request({
    "path": f"/automation/v4/flows/{flow_id}",
    "method": "GET",
})
flow = resp.json()
current_revision = flow["revisionId"]

# 2. Strip fields that cause validation errors on PUT
for field in ["createdAt", "updatedAt", "dataSources"]:
    flow.pop(field, None)

# 3. Make changes (e.g. rename, add action, toggle)
flow["name"] = "Renamed workflow"

# 4. PUT back — must include revisionId and type
flow["revisionId"] = current_revision  # ensure latest
resp = client.api_request({
    "path": f"/automation/v4/flows/{flow_id}",
    "method": "PUT",
    "body": flow,
})
```

### Required on PUT

- `revisionId` — must match the latest; if someone else edited the flow since your GET, the PUT will fail with a stale-revision error. Re-fetch and retry.
- `type` — `CONTACT_FLOW`, `COMPANY_FLOW`, `DEAL_FLOW`, `TICKET_FLOW`, etc. Don't change this mid-flight.

### Enabling / disabling a flow

```python
flow["isEnabled"] = True   # or False
# then PUT
```

## Creating a workflow

POST to `/automation/v4/flows` with a full flow definition. The shape is complex — the simplest path is to clone from an existing flow:

```python
# 1. Fetch an existing flow as a template
source = client.api_request({
    "path": f"/automation/v4/flows/{source_flow_id}",
    "method": "GET",
}).json()

# 2. Strip ID and revision
template = dict(source)
for field in ["id", "revisionId", "createdAt", "updatedAt", "dataSources"]:
    template.pop(field, None)
template["name"] = "New flow — cloned from " + source["name"]
template["isEnabled"] = False  # create disabled, enable after reviewing

# 3. POST to create
resp = client.api_request({
    "path": "/automation/v4/flows",
    "method": "POST",
    "body": template,
})
new_flow = resp.json()
print(f"Created flow {new_flow['id']}")
```

### Minimal contact flow from scratch

```python
{
    "type": "CONTACT_FLOW",
    "name": "Minimal test flow",
    "isEnabled": False,
    "enrollmentCriteria": {
        "type": "LIST_BASED",
        "listFilterBranch": {
            "filterBranchType": "AND",
            "filterBranches": [],
            "filters": [
                {
                    "filterType": "PROPERTY",
                    "property": "email",
                    "operation": {"operationType": "ALL_PROPERTY", "operator": "IS_KNOWN"},
                },
            ],
        },
        "unenrollObjects": False,
        "shouldReEnroll": False,
    },
    "actions": [],
    "suppressionListIds": [],
    "timeWindows": [],
    "blockedDates": [],
}
```

Real workflows have many more action types (SET_PROPERTY, DELAY, BRANCH, SEND_EMAIL, etc.). Always model on an existing flow from the target portal.

## Deleting / disabling

```python
# Disable (preferred — reversible, preserves enrolled records)
flow["isEnabled"] = False
# ... PUT

# Delete (irreversible)
client.api_request({
    "path": f"/automation/v4/flows/{flow_id}",
    "method": "DELETE",
})
```

## Enrollment and performance data

Enrollment history is available via the CRM timeline for contacts with the `contact-events` scope, but dedicated enrollment/performance endpoints are limited in v4. For detailed reporting, use the HubSpot UI's workflow performance tab or export via the Reports API.

## Mapping v3 workflow IDs to v4 flow IDs

If you have a v3 `workflowId`, translate to the v4 `flowId`:

```python
resp = client.api_request({
    "path": "/automation/v4/flows/migrate",
    "method": "POST",
    "body": {
        "v3WorkflowIds": [12345, 67890],
    },
})
# Response maps v3 ID -> v4 ID
```

## Custom-coded workflow actions

Custom code actions live inside flows but the code is edited through the UI. To read a flow's custom code action, inspect the action in the GET response — you'll find the code body under `actions[n].fields.body.value` (for JS) or similar.

To deploy a new version of a custom-code action's logic via API, you'd update the whole flow (PUT). For iterative development of the code itself, the UI is more practical.

### Custom workflow action definitions (for integrations)

If you're building a reusable workflow action (available to multiple customers via an app), that's a separate API — `/automation/v4/actions/{appId}`. Out of scope for most internal work; reach for it only when building a packaged Zapier-alternative integration.

## Common pitfalls

- **Revision conflicts**: two concurrent edits = one loses. Always refetch right before PUT.
- **Invalid custom code action request**: old or malformed code bodies fail validation. Inspect the error and sometimes the action needs to be recreated via UI first.
- **Classic vs visual flows**: some older flows have `isClassic: true`. These have different enrollment settings ("classic enrollment settings") and PUT may reject mixed shapes. Check `isClassic` on the GET response.
- **Cross-portal cloning silently breaks**: workflows reference property names, list IDs, owner IDs, and form GUIDs that don't exist in the target portal. After cloning, scan the full JSON for any numeric ID and validate it exists in the destination.
