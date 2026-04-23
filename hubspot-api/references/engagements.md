# Engagements

Engagements are records of activity: calls, emails, notes, meetings, and tasks. In the v3 API they're regular CRM objects (same patterns as contacts/deals), just with different object types and specific properties.

## Table of contents

- [Engagement object types](#engagement-object-types)
- [Logging a call](#logging-a-call)
- [Logging an email](#logging-an-email)
- [Creating a note](#creating-a-note)
- [Logging a meeting](#logging-a-meeting)
- [Creating a task](#creating-a-task)
- [Reading engagement history](#reading-engagement-history)
- [Common properties](#common-properties)

## Engagement object types

| Type | Endpoint | objectTypeId |
|---|---|---|
| Calls | `calls` | `0-48` |
| Emails | `emails` | `0-49` |
| Meetings | `meetings` | `0-47` |
| Notes | `notes` | `0-46` |
| Tasks | `tasks` | `0-27` |

All follow the CRM v3 object pattern: `/crm/v3/objects/{type}` with standard `basic_api`, `batch_api`, `search_api` endpoints via the SDK.

## Timestamps

All engagement timestamps (`hs_timestamp`, `hs_meeting_start_time`, etc.) use **epoch milliseconds**:

```python
from datetime import datetime, timezone
now_ms = int(datetime.now(tz=timezone.utc).timestamp() * 1000)
```

## Logging a call

```python
from hubspot.crm.objects.calls import SimplePublicObjectInputForCreate

call_input = SimplePublicObjectInputForCreate(
    properties={
        "hs_timestamp": str(int(datetime.now(tz=timezone.utc).timestamp() * 1000)),
        "hs_call_title": "Discovery call — Terrascope",
        "hs_call_body": "Discussed Q2 GTM scope. Johan to decide between Option A and B.",
        "hs_call_duration": "2700000",   # milliseconds — 45 min
        "hs_call_from_number": "+6591234567",
        "hs_call_to_number": "+6598765432",
        "hs_call_direction": "OUTBOUND",  # or INBOUND
        "hs_call_status": "COMPLETED",    # COMPLETED | NO_ANSWER | BUSY | FAILED | IN_PROGRESS | QUEUED | CANCELED
        "hs_call_recording_url": "",
        "hubspot_owner_id": "123456",     # owner ID, not user ID
    },
    associations=[
        {
            "to": {"id": contact_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 194}],  # call → contact
        },
        {
            "to": {"id": deal_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 206}],  # call → deal
        },
    ],
)

created = client.crm.objects.calls.basic_api.create(
    simple_public_object_input_for_create=call_input
)
print(f"Logged call {created.id}")
```

### Default association typeIds for calls

| From call → | typeId |
|---|---|
| Contact | 194 |
| Company | 182 |
| Deal | 206 |
| Ticket | 220 |

## Logging an email

```python
from hubspot.crm.objects.emails import SimplePublicObjectInputForCreate

email_input = SimplePublicObjectInputForCreate(
    properties={
        "hs_timestamp": str(now_ms),
        "hs_email_subject": "Re: Q2 GTM scope",
        "hs_email_html": "<p>Thanks for your time today…</p>",
        "hs_email_text": "Thanks for your time today…",       # plain-text fallback
        "hs_email_direction": "EMAIL",                         # EMAIL | INCOMING_EMAIL | FORWARDED_EMAIL
        "hs_email_status": "SENT",
        "hs_email_headers": '{"from": {"email": "dennis@work.flowers", "firstName": "Dennis"}, "to": [{"email": "johan@terrascope.earth"}]}',
        "hubspot_owner_id": "123456",
    },
    associations=[
        {
            "to": {"id": contact_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 198}],
        },
    ],
)
client.crm.objects.emails.basic_api.create(
    simple_public_object_input_for_create=email_input
)
```

### Default association typeIds for emails

| From email → | typeId |
|---|---|
| Contact | 198 |
| Company | 186 |
| Deal | 210 |
| Ticket | 224 |

`hs_email_headers` is a JSON-encoded string, not an object. Format as above.

## Creating a note

Simplest engagement — just body text and associations.

```python
from hubspot.crm.objects.notes import SimplePublicObjectInputForCreate

note_input = SimplePublicObjectInputForCreate(
    properties={
        "hs_timestamp": str(now_ms),
        "hs_note_body": "Followed up with Peter re: backend architecture. Next step: share scope doc.",
        "hubspot_owner_id": "123456",
    },
    associations=[
        {
            "to": {"id": contact_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 202}],
        },
    ],
)
client.crm.objects.notes.basic_api.create(
    simple_public_object_input_for_create=note_input
)
```

### Default association typeIds for notes

| From note → | typeId |
|---|---|
| Contact | 202 |
| Company | 190 |
| Deal | 214 |
| Ticket | 228 |

Notes support HTML in `hs_note_body`. Basic tags only: `<p>`, `<br>`, `<strong>`, `<em>`, `<ul>`, `<ol>`, `<li>`, `<a>`. Sanitise other markup before posting.

## Logging a meeting

```python
from hubspot.crm.objects.meetings import SimplePublicObjectInputForCreate

start_ms = int(datetime(2026, 4, 24, 10, 0, tzinfo=timezone.utc).timestamp() * 1000)
end_ms = int(datetime(2026, 4, 24, 11, 0, tzinfo=timezone.utc).timestamp() * 1000)

meeting_input = SimplePublicObjectInputForCreate(
    properties={
        "hs_timestamp": str(start_ms),
        "hs_meeting_title": "GTM workshop — Terrascope",
        "hs_meeting_body": "<p>Agenda: review scope, confirm pricing, agree timeline.</p>",
        "hs_meeting_start_time": str(start_ms),
        "hs_meeting_end_time": str(end_ms),
        "hs_meeting_location": "Google Meet",
        "hs_meeting_location_type": "VIRTUAL",      # VIRTUAL | PHYSICAL | PHONE
        "hs_meeting_outcome": "COMPLETED",          # SCHEDULED | COMPLETED | CANCELED | NO_SHOW | RESCHEDULED
        "hubspot_owner_id": "123456",
    },
    associations=[
        {
            "to": {"id": contact_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 200}],
        },
    ],
)
client.crm.objects.meetings.basic_api.create(
    simple_public_object_input_for_create=meeting_input
)
```

### Default association typeIds for meetings

| From meeting → | typeId |
|---|---|
| Contact | 200 |
| Company | 188 |
| Deal | 212 |
| Ticket | 226 |

## Creating a task

```python
from hubspot.crm.objects.tasks import SimplePublicObjectInputForCreate

due_ms = int(datetime(2026, 4, 25, 17, 0, tzinfo=timezone.utc).timestamp() * 1000)

task_input = SimplePublicObjectInputForCreate(
    properties={
        "hs_timestamp": str(due_ms),
        "hs_task_subject": "Send Terrascope revised proposal",
        "hs_task_body": "Incorporate feedback from Johan's last email and send by EOD Friday.",
        "hs_task_status": "NOT_STARTED",  # NOT_STARTED | IN_PROGRESS | COMPLETED | WAITING | DEFERRED
        "hs_task_priority": "HIGH",        # LOW | MEDIUM | HIGH
        "hs_task_type": "EMAIL",           # CALL | EMAIL | TODO | LINKED_IN | LINKED_IN_CONNECT | LINKED_IN_MESSAGE
        "hubspot_owner_id": "123456",
    },
    associations=[
        {
            "to": {"id": contact_id},
            "types": [{"associationCategory": "HUBSPOT_DEFINED", "associationTypeId": 204}],
        },
    ],
)
client.crm.objects.tasks.basic_api.create(
    simple_public_object_input_for_create=task_input
)
```

### Default association typeIds for tasks

| From task → | typeId |
|---|---|
| Contact | 204 |
| Company | 192 |
| Deal | 216 |
| Ticket | 230 |

## Reading engagement history

### All engagements on a record

The fastest pattern: read the associations for each engagement type separately.

```python
def get_all_engagements_for_contact(client, contact_id):
    engagements = {}
    for eng_type in ("calls", "emails", "meetings", "notes", "tasks"):
        resp = client.api_request({
            "path": f"/crm/v4/objects/contacts/{contact_id}/associations/{eng_type}",
            "method": "GET",
            "qs": {"limit": 500},
        })
        ids = [a["toObjectId"] for a in resp.json().get("results", [])]
        if ids:
            # Batch read to get properties
            api = getattr(client.crm.objects, eng_type).batch_api
            from hubspot.crm.objects import BatchReadInputSimplePublicObjectId
            req = BatchReadInputSimplePublicObjectId(
                inputs=[{"id": i} for i in ids],
                properties=_default_props_for(eng_type),
            )
            engagements[eng_type] = api.read(
                batch_read_input_simple_public_object_id=req
            ).results
        else:
            engagements[eng_type] = []
    return engagements

def _default_props_for(eng_type):
    return {
        "calls": ["hs_call_title", "hs_call_body", "hs_timestamp", "hs_call_duration", "hs_call_direction"],
        "emails": ["hs_email_subject", "hs_email_text", "hs_timestamp", "hs_email_direction"],
        "meetings": ["hs_meeting_title", "hs_meeting_body", "hs_meeting_start_time", "hs_meeting_end_time"],
        "notes": ["hs_note_body", "hs_timestamp"],
        "tasks": ["hs_task_subject", "hs_task_body", "hs_timestamp", "hs_task_status"],
    }[eng_type]
```

### Searching engagements

Same `search_api.do_search()` pattern as any other object. Example: find all uncompleted tasks owned by a user:

```python
from hubspot.crm.objects.tasks import PublicObjectSearchRequest

req = PublicObjectSearchRequest(
    filter_groups=[{
        "filters": [
            {"propertyName": "hubspot_owner_id", "operator": "EQ", "value": "123456"},
            {"propertyName": "hs_task_status", "operator": "NEQ", "value": "COMPLETED"},
        ]
    }],
    properties=["hs_task_subject", "hs_task_status", "hs_timestamp"],
    sorts=[{"propertyName": "hs_timestamp", "direction": "ASCENDING"}],
    limit=200,
)
resp = client.crm.objects.tasks.search_api.do_search(
    public_object_search_request=req
)
```

## Common properties

### On all engagements

| Property | Meaning |
|---|---|
| `hs_timestamp` | When it happened (epoch ms) — required |
| `hs_createdate` | When the record was created (read-only) |
| `hs_lastmodifieddate` | Read-only |
| `hubspot_owner_id` | Owner of the engagement (owner ID, not user ID) |

### Call-specific

- `hs_call_title`, `hs_call_body` — title and notes
- `hs_call_duration` — milliseconds
- `hs_call_direction` — `INBOUND` | `OUTBOUND`
- `hs_call_status` — `COMPLETED`, `NO_ANSWER`, `BUSY`, `FAILED`, `IN_PROGRESS`, `QUEUED`, `CANCELED`
- `hs_call_from_number`, `hs_call_to_number`
- `hs_call_recording_url`

### Email-specific

- `hs_email_subject`, `hs_email_text`, `hs_email_html`
- `hs_email_direction` — `EMAIL` (outbound), `INCOMING_EMAIL`, `FORWARDED_EMAIL`
- `hs_email_status` — `SENT`, `BOUNCED`, `FAILED`
- `hs_email_headers` — JSON-encoded string with from/to

### Meeting-specific

- `hs_meeting_title`, `hs_meeting_body`
- `hs_meeting_start_time`, `hs_meeting_end_time` — epoch ms
- `hs_meeting_location`, `hs_meeting_location_type` — `VIRTUAL`, `PHYSICAL`, `PHONE`
- `hs_meeting_outcome` — `SCHEDULED`, `COMPLETED`, `CANCELED`, `NO_SHOW`, `RESCHEDULED`

### Task-specific

- `hs_task_subject`, `hs_task_body`
- `hs_task_status` — `NOT_STARTED`, `IN_PROGRESS`, `COMPLETED`, `WAITING`, `DEFERRED`
- `hs_task_priority` — `LOW`, `MEDIUM`, `HIGH`
- `hs_task_type` — `CALL`, `EMAIL`, `TODO`, `LINKED_IN`, `LINKED_IN_CONNECT`, `LINKED_IN_MESSAGE`

### Gotchas

- **Owner IDs ≠ user IDs.** `hubspot_owner_id` is the owner ID from `/crm/v3/owners`, not the user settings ID. Fetch the mapping first if you have a user's email.
- **Associations are not optional in practice.** An engagement with no associations still exists but is effectively orphaned and won't show on any record timeline.
- **Historical emails**: the `emails` object logs individual email records; bulk marketing email sends are a different system — see `marketing.md`.
