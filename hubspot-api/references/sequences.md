# Sequences

Sequences are a Sales/Service Hub tool for timed, templated email outreach. The public API supports enrolling and unenrolling contacts. Creating/editing sequence content is UI-only.

## Table of contents

- [Required scope](#required-scope)
- [Prerequisites for enrolment](#prerequisites-for-enrolment)
- [Listing available sequences](#listing-available-sequences)
- [Enrolling a contact](#enrolling-a-contact)
- [Unenrolling a contact](#unenrolling-a-contact)
- [Common errors](#common-errors)
- [Bulk enrolment](#bulk-enrolment)

## Required scope

- `automation.sequences.enrollments.write` (for enroll/unenroll)
- `automation.sequences.enrollments.read` (for reading enrolment state)

The acting user behind the token must also:

- Hold a paid **Sales Hub or Service Hub** seat (Professional or Enterprise tier)
- Have a **connected personal inbox** (Gmail, Outlook, or IMAP)
- Have the Sequences permission in their HubSpot user permissions

**Private app gotcha:** when using a private app, the `userId` for enrolment must typically be the owner-user who created the private app, and that user must meet all the above requirements. Team-shared inboxes don't count.

## Prerequisites for enrolment

Before calling the enrolment endpoint, verify:

1. Contact has a valid email property
2. The sender user has a connected inbox (`GET /email/public/v1/settings/accounts/{userId}` or check in UI)
3. Contact is not already enrolled in another sequence (contacts can only be in one at a time)
4. The sequence exists and is active

## Listing available sequences

```python
resp = client.api_request({
    "path": "/automation/sequences/2026-03/sequences",
    "method": "GET",
})
sequences = resp.json().get("results", [])
for s in sequences:
    print(s["id"], s["name"])
```

## Enrolling a contact

### Endpoint

`POST /automation/sequences/2026-03/enrollments?userId={userId}`

The `userId` query parameter is the ID of the HubSpot user doing the enrolment — this is the **owner-user ID** (from `/crm/v3/owners`), **not** the settings user ID. The sender email must be connected to this user.

```python
def get_owner_user_id(client, email):
    """Return the owner-user ID for a given email, for use as userId in sequences."""
    resp = client.api_request({
        "path": "/crm/v3/owners",
        "method": "GET",
        "qs": {"email": email, "limit": 100},
    })
    results = resp.json().get("results", [])
    if not results:
        raise ValueError(f"No owner found for {email}")
    return results[0]["userId"]  # userId field on the owner record
```

### Enrolment call

```python
def enrol_contact(client, user_id, contact_id, sequence_id, sender_email, sender_alias=None):
    body = {
        "contactId": str(contact_id),
        "senderEmail": sender_email,
        "sequenceId": str(sequence_id),
    }
    if sender_alias:
        body["senderAliasAddress"] = sender_alias

    resp = client.api_request({
        "path": "/automation/sequences/2026-03/enrollments",
        "method": "POST",
        "qs": {"userId": str(user_id)},
        "body": body,
    })
    if resp.status_code >= 400:
        raise RuntimeError(f"Enrolment failed ({resp.status_code}): {resp.text}")
    return resp.json()
```

Response:

```json
{
  "id": "enrollment-id",
  "toEmail": "contact@example.com",
  "enrolledAt": "2026-04-23T10:00:00Z",
  "updatedAt": "2026-04-23T10:00:00Z"
}
```

## Unenrolling a contact

```python
def unenrol(client, enrollment_id):
    resp = client.api_request({
        "path": f"/automation/sequences/2026-03/enrollments/{enrollment_id}/cancel",
        "method": "POST",
    })
    if resp.status_code >= 400:
        raise RuntimeError(f"Unenrol failed: {resp.text}")
```

Or find the active enrolment for a contact and cancel it:

```python
def unenrol_contact(client, user_id, contact_id):
    # List active enrolments for the contact
    resp = client.api_request({
        "path": "/automation/sequences/2026-03/enrollments",
        "method": "GET",
        "qs": {"userId": str(user_id), "contactId": str(contact_id)},
    })
    for e in resp.json().get("results", []):
        if e.get("status") == "ACTIVE":
            unenrol(client, e["id"])
```

## Common errors

| HTTP | Body contains | Meaning | Fix |
|---|---|---|---|
| 500 | "User X for Portal Y has no connected inboxes" | The acting user's inbox isn't connected | Connect personal email in HubSpot settings; verify in UI |
| 500 | generic | Frequently caused by seat/permission mismatch | Confirm user has Sales/Service seat and Sequences permission |
| 400 | "A contact with the same email as the user" | The contact being enrolled has the same email as the sender | Use a different sender email or contact |
| 400 | "Contact is already enrolled" | Contacts can only be in one sequence at a time | Unenrol from the existing sequence first |
| 403 | `MISSING_SCOPES` | Token missing `automation.sequences.enrollments.write` | Add scope in private app settings |
| 429 | rate limit | Daily send limits on the user are separate from API rate limits | Slow down; max ~100 enrolments per user per day is typical |

## Bulk enrolment

Sequence enrolment is **per-user-per-day rate-limited** by the connected inbox's sending limits, not just the API rate limit. Plan accordingly:

- Google Workspace: ~500 emails/day per user (hard cap)
- Microsoft 365: ~300–500/day
- HubSpot's official guidance: don't bulk-enrol more than 100 contacts via API in a single burst

```python
import time

def bulk_enrol(client, user_id, contact_ids, sequence_id, sender_email,
               delay_seconds=1.0):
    succeeded, failed = [], []
    for cid in contact_ids:
        try:
            result = enrol_contact(client, user_id, cid, sequence_id, sender_email)
            succeeded.append({"contact_id": cid, "enrollment_id": result.get("id")})
        except Exception as e:
            failed.append({"contact_id": cid, "error": str(e)})
        time.sleep(delay_seconds)
    return succeeded, failed
```

Always dry-run first and flag high-risk scenarios to the user (e.g. "you're about to enrol 347 contacts; typical daily inbox send limit is ~500 — proceed?").

## Alternatives to API enrolment

If the API path is blocked (seat issues, scope limits), workflows can enrol contacts in sequences natively — no API call needed. See `workflows.md` for building enrolment via a workflow trigger, which bypasses the per-user inbox requirement in some configurations but requires Sales/Service Hub Enterprise.
