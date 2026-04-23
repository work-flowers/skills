# Core Patterns

The patterns every script needs: pagination, rate limiting, batch operations, error handling, and the `api_request` escape hatch.

## Table of contents

- [Standard script scaffold](#standard-script-scaffold)
- [Pagination](#pagination)
- [Batch operations](#batch-operations)
- [Rate limiting and retries](#rate-limiting-and-retries)
- [Error handling](#error-handling)
- [The `api_request` escape hatch](#the-api_request-escape-hatch)
- [Exporting to CSV](#exporting-to-csv)
- [Dry-run pattern](#dry-run-pattern)

## Standard script scaffold

Every script starts like this:

```python
"""
<One-line description of what this script does>
Portal: <Terrascope | Knoxx | ...>
"""
import os
import sys
import time
from hubspot import HubSpot
from urllib3.util.retry import Retry

# Retry on 429 and 5xx with backoff
retry = Retry(
    total=5,
    backoff_factor=0.3,
    status_forcelist=(429, 500, 502, 503, 504),
    allowed_methods=frozenset(["GET", "POST", "PUT", "PATCH", "DELETE"]),
)

token = os.environ.get("HUBSPOT_ACCESS_TOKEN")
if not token:
    sys.exit("HUBSPOT_ACCESS_TOKEN not set")

client = HubSpot(access_token=token, retry=retry)
```

## Pagination

### `basic_api.get_page()` — for `contacts`, `companies`, `deals`, `tickets`, custom objects

Max 100 per page. Paginate by following `paging.next.after`:

```python
def get_all_contacts(client, properties=None):
    results = []
    after = None
    while True:
        page = client.crm.contacts.basic_api.get_page(
            limit=100,
            after=after,
            properties=properties or [],
            archived=False,
        )
        results.extend(page.results)
        if not page.paging or not page.paging.next:
            break
        after = page.paging.next.after
    return results
```

### `search_api.do_search()` — with filters

Max 200 per page. The SDK's `do_search()` returns one page; paginate manually via the request body's `after`:

```python
from hubspot.crm.contacts import PublicObjectSearchRequest

def search_all_contacts(client, filter_groups, properties=None):
    results = []
    after = 0
    while True:
        req = PublicObjectSearchRequest(
            filter_groups=filter_groups,
            properties=properties or [],
            limit=200,
            after=after,
        )
        page = client.crm.contacts.search_api.do_search(
            public_object_search_request=req
        )
        results.extend(page.results)
        if not page.paging or not page.paging.next:
            break
        after = int(page.paging.next.after)
        if after >= 10000:
            # Search API hard cap
            break
    return results
```

### Breaking the 10,000-record search cap

The Search API refuses to return results past record 10,000. Work around by paginating on an indexed property like `hs_object_id`:

```python
def search_all_past_10k(client, base_filters, properties=None):
    """
    Break a large search into chunks by hs_object_id.
    base_filters: list of additional filter groups (AND'd with the id range).
    """
    results = []
    last_id = 0
    while True:
        id_filter = {
            "filters": [{"propertyName": "hs_object_id", "operator": "GT", "value": str(last_id)}]
        }
        # Merge: every filter group gets the id constraint AND'd in
        merged_groups = [
            {"filters": g.get("filters", []) + id_filter["filters"]}
            for g in (base_filters or [{"filters": []}])
        ]
        req = PublicObjectSearchRequest(
            filter_groups=merged_groups,
            properties=properties or [],
            sorts=[{"propertyName": "hs_object_id", "direction": "ASCENDING"}],
            limit=200,
        )
        page = client.crm.contacts.search_api.do_search(
            public_object_search_request=req
        )
        if not page.results:
            break
        results.extend(page.results)
        last_id = int(page.results[-1].id)
        if len(page.results) < 200:
            break
    return results
```

## Batch operations

Batch endpoints take up to **100 records per call**. Always chunk.

### Batch read by ID

```python
from hubspot.crm.contacts import BatchReadInputSimplePublicObjectId

def batch_read_contacts(client, contact_ids, properties):
    all_results = []
    for i in range(0, len(contact_ids), 100):
        chunk = contact_ids[i:i + 100]
        req = BatchReadInputSimplePublicObjectId(
            inputs=[{"id": str(cid)} for cid in chunk],
            properties=properties,
        )
        resp = client.crm.contacts.batch_api.read(
            batch_read_input_simple_public_object_id=req
        )
        all_results.extend(resp.results)
    return all_results
```

### Batch update

```python
from hubspot.crm.contacts import BatchInputSimplePublicObjectBatchInput

def batch_update_contacts(client, updates):
    """
    updates: list of dicts like {"id": "123", "properties": {"lifecyclestage": "lead"}}
    """
    successes, failures = [], []
    for i in range(0, len(updates), 100):
        chunk = updates[i:i + 100]
        req = BatchInputSimplePublicObjectBatchInput(inputs=chunk)
        try:
            resp = client.crm.contacts.batch_api.update(
                batch_input_simple_public_object_batch_input=req
            )
            successes.extend(resp.results)
        except Exception as e:
            failures.append({"chunk_start": i, "error": str(e)})
    return successes, failures
```

### Batch upsert (by unique property)

```python
from hubspot.crm.contacts import BatchInputSimplePublicObjectBatchInputUpsert

def batch_upsert_contacts(client, upserts, id_property="email"):
    """
    upserts: [{"idProperty": "email", "id": "x@y.com", "properties": {...}}, ...]
    """
    all_results = []
    for i in range(0, len(upserts), 100):
        chunk = upserts[i:i + 100]
        req = BatchInputSimplePublicObjectBatchInputUpsert(inputs=chunk)
        resp = client.crm.contacts.batch_api.upsert(
            batch_input_simple_public_object_batch_input_upsert=req
        )
        all_results.extend(resp.results)
    return all_results
```

## Rate limiting and retries

The `Retry` config in the scaffold handles 429s and 5xx automatically. For additional safety on bulk jobs:

```python
import time

def sleep_between_calls(seconds=0.1):
    """
    Call between loops in bulk jobs. At 190 req/10s limit,
    10 req/s is safe; 0.1s sleep is a cushion.
    """
    time.sleep(seconds)

# Search API: max 5 req/s. Enforce 0.25s between search calls.
def sleep_search():
    time.sleep(0.25)
```

Check remaining quota periodically:

```python
def check_rate_limit(client):
    """
    The SDK doesn't expose response headers cleanly; use api_request for this.
    Returns dict with remaining/max for the current window.
    """
    resp = client.api_request({"path": "/crm/v3/objects/contacts", "qs": {"limit": 1}})
    return {
        "daily_remaining": resp.headers.get("X-HubSpot-RateLimit-Daily-Remaining"),
        "daily_max": resp.headers.get("X-HubSpot-RateLimit-Daily"),
        "ten_sec_remaining": resp.headers.get("X-HubSpot-RateLimit-Remaining"),
    }
```

## Error handling

```python
from hubspot.crm.contacts import ApiException

try:
    result = client.crm.contacts.basic_api.get_by_id(contact_id="123")
except ApiException as e:
    # e.status (HTTP code), e.body (response body), e.reason
    if e.status == 404:
        print("Contact not found")
    elif e.status == 401:
        print("Token invalid or missing scopes")
    elif e.status == 429:
        print("Rate limit hit — Retry should have caught this")
    else:
        print(f"HubSpot error {e.status}: {e.body}")
```

### Common error bodies

- `MISSING_SCOPES` — add the required scope to the private app in HubSpot UI
- `VALIDATION_ERROR` — check property names, required fields, field type mismatches
- `OBJECT_NOT_FOUND` — ID doesn't exist or is archived
- `PROPERTY_DOESNT_EXIST` — case-sensitive; use the internal name, not the label
- `RATE_LIMITS` — too many requests; the Retry config should catch most

## The `api_request` escape hatch

Not every endpoint is wrapped in the SDK (e.g. workflows v4, lists v3 membership join-order, some sequence endpoints). Use `client.api_request()` for raw calls:

```python
# GET with query params
resp = client.api_request({
    "path": "/automation/v4/flows",
    "method": "GET",
    "qs": {"limit": 100},
})
data = resp.json()

# POST with body
resp = client.api_request({
    "path": "/crm/v3/lists",
    "method": "POST",
    "body": {
        "name": "My list",
        "objectTypeId": "0-1",
        "processingType": "MANUAL",
    },
})
data = resp.json()

# Handle errors manually on api_request
if resp.status_code >= 400:
    raise RuntimeError(f"{resp.status_code}: {resp.text}")
```

`api_request` returns a standard `requests.Response` object — use `resp.status_code`, `resp.text`, `resp.json()`, and `resp.headers` as with any `requests` call. It respects the client's `retry` configuration and auth, but does NOT raise on 4xx/5xx — check `resp.status_code` manually.

## Exporting to CSV

```python
import csv
from pathlib import Path

def export_to_csv(records, fieldnames, path):
    """
    records: list of dicts (e.g. [r.properties for r in results])
    fieldnames: ordered list of columns
    path: pathlib.Path or string. Save to a location accessible to the user
          in the current environment — the caller decides the convention
          (Cowork workspace folder, Claude Code cwd, claude.ai outputs dir).
    """
    Path(path).parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=fieldnames, extrasaction="ignore")
        writer.writeheader()
        for r in records:
            writer.writerow(r)
    print(f"Wrote {len(records)} rows to {path}")
```

## Dry-run pattern

For any bulk write, compute the diff first, show a summary, then execute only on confirmation.

```python
def dry_run_summary(updates, label="updates"):
    """
    updates: list of {"id": ..., "properties": {...}}
    """
    print(f"\n=== DRY RUN: {len(updates)} {label} ===")
    for u in updates[:5]:
        print(f"  {u['id']}: {u['properties']}")
    if len(updates) > 5:
        print(f"  ... and {len(updates) - 5} more")
    print()
```

Pattern in a script:

```python
updates = compute_updates(...)  # build the list of changes
dry_run_summary(updates, "contact updates")

# In interactive use, wait for confirmation. In non-interactive runs,
# control with an env var or CLI flag:
if os.environ.get("EXECUTE") != "1":
    print("Set EXECUTE=1 to apply these changes.")
    sys.exit(0)

batch_update_contacts(client, updates)
```
