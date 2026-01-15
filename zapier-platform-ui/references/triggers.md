# Triggers Reference

Detailed patterns for polling and REST Hook triggers in Zapier Platform UI.

## Polling Triggers

Zapier polls your endpoint on a schedule (1-15 minutes based on user's plan).

### Settings Configuration
- **Key**: `new_contact` (use `new_` or `updated_` prefix)
- **Name**: "New Contact" or "Updated Contact"
- **Noun**: "Contact"
- **Description**: "Triggers when a new contact is created." (Starts with "Triggers when")
- **Visibility**: Shown (or Hidden for dropdown-only triggers)

### Input Designer
Add optional filters:
```
- status (Dropdown) - "Only trigger for contacts with this status"
  Options: All, Active, Inactive
  Default: All

- list_id (Dynamic Dropdown) - "Only trigger for contacts in this list"
```

### API Configuration

**Basic polling:**
```
Method: GET
URL: https://api.example.com/contacts
Params:
  sort: created_at
  order: desc
  limit: 100
```

**With filters from input:**
```
URL: https://api.example.com/contacts
Params:
  sort: created_at
  order: desc
  limit: 100
  status: {{bundle.inputData.status}}
  list_id: {{bundle.inputData.list_id}}
```

### Response Requirements

**Must return array sorted newest-first:**
```json
[
  {"id": "3", "email": "newest@test.com", "created_at": "2025-01-14T12:00:00Z"},
  {"id": "2", "email": "middle@test.com", "created_at": "2025-01-14T11:00:00Z"},
  {"id": "1", "email": "oldest@test.com", "created_at": "2025-01-14T10:00:00Z"}
]
```

**Each item must have unique `id` field** for deduplication:
- Zapier remembers IDs and skips items it's seen before
- If your API doesn't return `id`, use Code Mode to add one

### Code Mode - Add ID Field
```javascript
return z.request(options).then(response => {
  return response.data.contacts.map(contact => ({
    id: contact.contact_id, // Map API's ID field to 'id'
    ...contact
  }));
});
```

### Code Mode - Sort Response
```javascript
return z.request(options).then(response => {
  // Sort by created_at descending if API doesn't
  return response.data.sort((a, b) => 
    new Date(b.created_at) - new Date(a.created_at)
  );
});
```

---

## REST Hook Triggers (Instant)

Your app sends webhooks to Zapier when events occur.

### Prerequisites
Your API must support:
1. **Subscribe endpoint**: Register webhook URL
2. **Unsubscribe endpoint**: Remove webhook when Zap turns off
3. **Webhook delivery**: POST data to the registered URL

### Settings Configuration
- **Key**: `new_contact_instant`
- **Name**: "New Contact (Instant)"
- **Noun**: "Contact"  
- **Description**: "Triggers instantly when a new contact is created."

### API Configuration - REST Hook Type

Select "REST Hook" as trigger type, then configure:

**1. Subscribe Request**
Registers Zapier's webhook URL with your app:
```
Method: POST
URL: https://api.example.com/webhooks

Request Body:
{
  "target_url": "{{bundle.targetUrl}}",
  "event": "contact.created"
}
```

Response should include webhook ID for later unsubscribe:
```json
{"id": "webhook_123", "status": "active"}
```

**2. Unsubscribe Request**
Removes webhook when Zap turns off:
```
Method: DELETE
URL: https://api.example.com/webhooks/{{bundle.subscribeData.id}}
```

**3. Perform (Webhook Handler)**
Processes incoming webhook—usually just returns the data:
```javascript
return [bundle.cleanedRequest];
```

**4. Perform List (Sample Data)**
GET endpoint for testing and sample data:
```
Method: GET
URL: https://api.example.com/contacts?limit=1&sort=created_at&order=desc
```

**Critical**: Response schema must match webhook payload schema exactly.

### Code Mode - Subscribe
```javascript
const options = {
  url: 'https://api.example.com/webhooks',
  method: 'POST',
  body: {
    target_url: bundle.targetUrl,
    event_types: ['contact.created'],
    // Include any auth/identifying info your API needs
    workspace_id: bundle.authData.workspace_id
  }
};

return z.request(options).then(response => {
  // Store webhook ID for unsubscribe
  return {
    id: response.data.webhook_id
  };
});
```

### Code Mode - Unsubscribe
```javascript
const options = {
  url: `https://api.example.com/webhooks/${bundle.subscribeData.id}`,
  method: 'DELETE'
};

return z.request(options).then(response => {
  return response.data;
});
```

### Code Mode - Perform (Webhook Handler)
```javascript
// Transform incoming webhook data if needed
const payload = bundle.cleanedRequest;

return [{
  id: payload.data.id,
  email: payload.data.attributes.email,
  name: `${payload.data.attributes.first_name} ${payload.data.attributes.last_name}`,
  created_at: payload.data.attributes.created_at
}];
```

### Code Mode - Perform List
```javascript
const options = {
  url: 'https://api.example.com/contacts',
  params: {
    limit: 3,
    sort: '-created_at'  // Newest first
  }
};

return z.request(options).then(response => {
  // Transform to match webhook payload structure
  return response.data.contacts.map(contact => ({
    id: contact.id,
    email: contact.email,
    name: `${contact.first_name} ${contact.last_name}`,
    created_at: contact.created_at
  }));
});
```

---

## Deduplication

### For Polling Triggers
Zapier uses `id` field to skip items it's already processed.

**Default behavior:**
- First poll: Items 1, 2, 3 trigger Zap
- Second poll: Items 3, 4 returned → Only item 4 triggers (3 already seen)

**Custom ID field:**
If your API uses different field name, map it in Code Mode:
```javascript
return response.data.map(item => ({
  id: item.record_id,  // Use record_id as dedup key
  ...item
}));
```

**Compound IDs:**
For items without natural unique ID:
```javascript
return response.data.map(item => ({
  id: `${item.date}_${item.email}_${item.action}`,
  ...item
}));
```

### For REST Hook Triggers
No automatic deduplication—your app controls what triggers.

---

## Updated Item Triggers

For triggers that watch for changes to existing records.

### Settings
- **Name**: "Updated Contact"
- **Description**: "Triggers when a contact is updated."

### API Requirements
Endpoint must:
- Return recently modified items (not just created)
- Sort by `updated_at` descending
- Include `id` field (same item, different updates = same ID)

### Help Users Understand Updates
Add description explaining what counts as "updated":
```
"Triggers when a contact's email, name, or status changes. 
Does not trigger for note additions or activity logs."
```

### Code Mode - Filter Update Types
```javascript
return z.request(options).then(response => {
  // Only trigger for specific update types
  return response.data.filter(item => 
    item.updated_fields && 
    item.updated_fields.some(f => ['email', 'status', 'name'].includes(f))
  );
});
```

---

## Hidden Triggers for Dropdowns

Create triggers that only power dynamic dropdowns, not user-facing Zaps.

### Settings
- **Name**: "List Projects"
- **Visibility**: Hidden

### API Configuration
```
Method: GET
URL: https://api.example.com/projects?limit=100
```

### Use in Dynamic Dropdown
In another trigger/action's Input Designer:
1. Add field (e.g., `project_id`)
2. Enable "Dynamic Dropdown"
3. Select "List Projects" trigger
4. Set Value field: `id`
5. Set Label field: `name`

---

## Sample Data & Output Fields

### Define Output Fields
In Output Fields section, specify fields Zapier shows in subsequent steps:
```
id (string) - Unique contact ID
email (string) - Contact email address
first_name (string) - First name
last_name (string) - Last name
created_at (datetime) - When contact was created
```

### Provide Sample Data
Add representative sample for users setting up Zaps:
```json
{
  "id": "contact_123",
  "email": "sample@example.com",
  "first_name": "Jane",
  "last_name": "Doe",
  "created_at": "2025-01-14T10:00:00Z"
}
```

---

## Testing Triggers

### Polling Triggers
1. Connect test account
2. Add test input data
3. Click "Test Your Request"
4. Verify response is array, sorted correctly, has `id` field

### REST Hook Triggers
1. Create Zap with your trigger
2. Turn Zap on
3. Perform trigger action in your app
4. Check Zap history for triggered run
5. Turn Zap off
6. Verify webhook unsubscribed in your app

### Common Issues
- **No results**: Check endpoint URL, filters, auth
- **Duplicate triggers**: Verify `id` field is unique
- **Missing data**: Check response mapping in Code Mode
- **Webhook not received**: Verify subscribe saved correct URL
