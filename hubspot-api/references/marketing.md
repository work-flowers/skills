# Marketing (Emails, Campaigns, Forms)

HubSpot's marketing APIs cover single-send (transactional), marketing email management, campaigns, forms, and subscription preferences. These are more fragmented than the CRM API — different versions, different scopes, occasional legacy endpoints.

## Table of contents

- [Required scopes](#required-scopes)
- [Single-send API (transactional emails)](#single-send-api-transactional-emails)
- [Marketing emails](#marketing-emails)
- [Campaigns](#campaigns)
- [Forms](#forms)
- [Subscription preferences](#subscription-preferences)

## Required scopes

- `transactional-email` — single-send API
- `content` — marketing email read/write
- `marketing.campaigns.read`, `marketing.campaigns.write` — campaigns
- `forms` — forms read/write
- `crm.objects.contacts.write` + `communication_preferences.read_write` — subscription management

## Single-send API (transactional emails)

The single-send API sends one-off emails based on a designed HubSpot email template. Requires **Marketing Hub Enterprise** or the **Transactional Email** add-on.

### Endpoint

`POST /marketing/v3/transactional/single-email/send`

### Example

```python
resp = client.api_request({
    "path": "/marketing/v3/transactional/single-email/send",
    "method": "POST",
    "body": {
        "emailId": 12345678,  # template email ID — get from marketing email list
        "message": {
            "to": "contact@example.com",
            "from": "Dennis Chiuten <dennis@work.flowers>",
            "sendId": "unique-dedup-id-optional",
        },
        "contactProperties": {
            "firstname": "Johan",
            "lastname": "Terrascope",
        },
        "customProperties": {
            "order_number": "ORD-12345",
            "total_amount": "$50,000",
        },
    },
})
```

- `emailId` must be an email in HubSpot marked "Save for single-send via API"
- `contactProperties` create/update the contact record
- `customProperties` are available as merge tags in the template body (e.g. `{{ custom.order_number }}`)

## Marketing emails

### List marketing emails

```python
resp = client.api_request({
    "path": "/marketing/v3/emails/",
    "method": "GET",
    "qs": {"limit": 100, "state": "PUBLISHED"},
})
emails = resp.json().get("results", [])
```

States: `DRAFT`, `PUBLISHED`, `AUTOMATED`, `SCHEDULED`.

### Get email statistics

Per-email performance stats (opens, clicks, bounces):

```python
resp = client.api_request({
    "path": f"/marketing/v3/emails/statistics/list",
    "method": "GET",
    "qs": {
        "startTimestamp": "2026-01-01T00:00:00Z",
        "endTimestamp": "2026-04-01T00:00:00Z",
        "emailIds": "12345678,23456789",
    },
})
```

### Clone a marketing email

```python
resp = client.api_request({
    "path": f"/marketing/v3/emails/clone",
    "method": "POST",
    "body": {
        "id": str(source_email_id),
        "cloneName": "Cloned email name",
    },
})
new_email = resp.json()
```

### Publish / schedule an email

Use the UI for one-off publishes. For programmatic scheduling:

```python
resp = client.api_request({
    "path": f"/marketing/v3/emails/{email_id}",
    "method": "PATCH",
    "body": {
        "publishDate": "2026-05-01T09:00:00+08:00",
    },
})
```

## Campaigns

Campaigns group marketing assets (emails, social posts, landing pages) under a single initiative for attribution. Requires Marketing Hub Professional+.

### List campaigns

```python
resp = client.api_request({
    "path": "/marketing/v3/campaigns/",
    "method": "GET",
    "qs": {"limit": 100},
})
```

### Create a campaign

```python
resp = client.api_request({
    "path": "/marketing/v3/campaigns/",
    "method": "POST",
    "body": {
        "properties": {
            "hs_name": "Q2 2026 Terrascope Launch",
            "hs_start_date": "2026-05-01",
            "hs_end_date": "2026-06-30",
            "hs_goal": "Generate 50 MQLs",
            "hs_budget_total": 15000,
            "hs_currency_code": "SGD",
        },
    },
})
campaign_id = resp.json()["id"]
```

### Associate an asset to a campaign

Asset types include `FORM`, `MARKETING_EMAIL`, `LANDING_PAGE`, `SOCIAL_POST`, etc. Get the full list via:

```python
resp = client.api_request({
    "path": "/marketing/v3/campaigns/asset-types",
    "method": "GET",
})
```

Then associate:

```python
client.api_request({
    "path": f"/marketing/v3/campaigns/{campaign_id}/assets/{asset_type}/{asset_id}",
    "method": "PUT",
})
```

### Campaign metrics

```python
resp = client.api_request({
    "path": f"/marketing/v3/campaigns/{campaign_id}/metrics",
    "method": "GET",
    "qs": {
        "startDate": "2026-05-01",
        "endDate": "2026-06-30",
    },
})
```

### Contacts influenced by a campaign

```python
def get_campaign_contacts(client, campaign_id, attribution_type="INFLUENCED"):
    contacts = []
    after = None
    while True:
        qs = {"limit": 100, "attributionType": attribution_type}
        if after:
            qs["after"] = after
        resp = client.api_request({
            "path": f"/marketing/v3/campaigns/{campaign_id}/contacts",
            "method": "GET",
            "qs": qs,
        })
        data = resp.json()
        contacts.extend(data.get("results", []))
        paging = data.get("paging", {}).get("next", {})
        if not paging:
            break
        after = paging.get("after")
    return contacts
```

Attribution types: `INFLUENCED` (any touchpoint) | `FIRST_TOUCH` | `LAST_TOUCH`.

## Forms

Two form systems coexist:

1. **Marketing forms** — legacy, still most common. `/marketing/v3/forms/`
2. **CRM forms** — newer, object-linked. Same base endpoint with different shapes.

### List forms

```python
resp = client.api_request({
    "path": "/marketing/v3/forms/",
    "method": "GET",
    "qs": {"limit": 100},
})
```

### Get a form (full definition including fields)

```python
resp = client.api_request({
    "path": f"/marketing/v3/forms/{form_id}",
    "method": "GET",
})
form = resp.json()
# form["fieldGroups"] contains the field structure
```

### Submit data to a form (lead capture)

The submission endpoint is **different from the CRM API** and does NOT require authentication (it's the same endpoint the public form uses). For server-side submission from an authenticated context:

```python
import requests

resp = requests.post(
    f"https://api.hsforms.com/submissions/v3/integration/submit/{portal_id}/{form_id}",
    json={
        "fields": [
            {"name": "email", "value": "contact@example.com"},
            {"name": "firstname", "value": "Johan"},
            {"name": "company", "value": "Terrascope"},
        ],
        "context": {
            "hutk": "optional_hubspot_usertoken_from_cookie",
            "pageUri": "https://work.flowers/contact",
            "pageName": "Contact form",
        },
        "legalConsentOptions": {
            "consent": {
                "consentToProcess": True,
                "text": "I agree to allow work.flowers to store and process my personal data.",
                "communications": [
                    {
                        "value": True,
                        "subscriptionTypeId": 999,
                        "text": "I agree to receive marketing communications from work.flowers.",
                    }
                ],
            }
        },
    },
)
```

Portal ID is the Hub ID — find via `GET /integrations/v1/me` or in the HubSpot URL.

### Create a marketing form

```python
resp = client.api_request({
    "path": "/marketing/v3/forms/",
    "method": "POST",
    "body": {
        "formType": "hubspot",
        "name": "Contact us",
        "fieldGroups": [
            {
                "groupType": "default_group",
                "richTextType": "text",
                "fields": [
                    {
                        "objectTypeId": "0-1",
                        "name": "email",
                        "fieldType": "email",
                        "label": "Your email",
                        "required": True,
                    },
                    {
                        "objectTypeId": "0-1",
                        "name": "firstname",
                        "fieldType": "single_line_text",
                        "label": "First name",
                        "required": False,
                    },
                ],
            }
        ],
        "configuration": {
            "language": "en",
            "cloneable": True,
            "editable": True,
        },
    },
})
```

## Subscription preferences

Subscription types are the marketing categories contacts can opt in/out of (e.g. "Product updates", "Weekly newsletter").

### List subscription types

```python
resp = client.api_request({
    "path": "/communication-preferences/v3/definitions",
    "method": "GET",
})
subscriptions = resp.json().get("subscriptionDefinitions", [])
for s in subscriptions:
    print(s["id"], s["name"], s["description"])
```

### Get a contact's subscription status

```python
resp = client.api_request({
    "path": f"/communication-preferences/v3/status/email/{contact_email}",
    "method": "GET",
})
```

### Update a contact's subscription

```python
# Opt in
client.api_request({
    "path": f"/communication-preferences/v3/subscribe",
    "method": "POST",
    "body": {
        "emailAddress": contact_email,
        "subscriptionId": str(subscription_id),
        "legalBasis": "LEGITIMATE_INTEREST_CLIENT",
        "legalBasisExplanation": "Opted in on contact form submission",
    },
})

# Opt out
client.api_request({
    "path": f"/communication-preferences/v3/unsubscribe",
    "method": "POST",
    "body": {
        "emailAddress": contact_email,
        "subscriptionId": str(subscription_id),
    },
})
```

### Unsubscribe from all (GDPR)

```python
client.api_request({
    "path": f"/communication-preferences/v3/unsubscribe-all",
    "method": "POST",
    "body": {"emailAddress": contact_email},
})
```

## Common gotchas

- **Marketing emails vs engagement emails**: the `emails` object under `/crm/v3/objects/emails` is the engagement record of an email sent/received. Marketing emails under `/marketing/v3/emails/` are the templates and sends managed by the marketing team. Don't conflate.
- **Single-send templates must be enabled for API use** — in the marketing email editor, flip the "Save for single-send via API" toggle before trying to send via API.
- **Form submissions don't require auth** — the `hsforms.com` submission endpoint is public. Don't put your private app token in the request body.
- **Subscription IDs are per-portal** — when migrating automations across portals, you must re-map subscription IDs. Default subscription IDs (for the auto-generated "One to one" type) may also differ by portal age.
- **Campaign asset-types list is authoritative** — new types get added periodically. Always call `/campaigns/asset-types` rather than hard-coding.
