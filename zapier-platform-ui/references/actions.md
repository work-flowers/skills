# Actions Reference

Detailed patterns for Create, Search, and Update actions in Zapier Platform UI.

## Create Actions

Add new records to your app.

### Settings Configuration
- **Key**: Unique identifier (e.g., `create_contact`)
- **Name**: Title case with action verb (e.g., "Create Contact")
- **Noun**: Singular noun (e.g., "Contact")
- **Description**: Starts with "Creates a new" (e.g., "Creates a new contact in your CRM.")

### Input Designer
Add fields for all data your API accepts:

```
Required fields first:
- email (String, Required) - "Contact's email address"
- first_name (String, Required) - "Contact's first name"

Optional fields after:
- last_name (String) - "Contact's last name"  
- phone (String) - "Phone number with country code"
- company (String) - "Company name"
- tags (String, Allows Multiples) - "Tags to apply to this contact"
```

### API Configuration
```
Method: POST
URL: https://api.example.com/contacts

Request Body (JSON):
{
  "email": "{{bundle.inputData.email}}",
  "first_name": "{{bundle.inputData.first_name}}",
  "last_name": "{{bundle.inputData.last_name}}",
  "phone": "{{bundle.inputData.phone}}",
  "company": "{{bundle.inputData.company}}",
  "tags": {{bundle.inputData.tags}}
}
```

### Expected Response
Return the complete created object:
```json
{
  "id": "contact_123",
  "email": "user@example.com",
  "first_name": "Jane",
  "last_name": "Doe",
  "created_at": "2025-01-14T10:00:00Z",
  "url": "https://app.example.com/contacts/contact_123"
}
```

**Never return only:**
```json
{"success": true}  // Bad - no data for subsequent Zap steps
```

### Code Mode Example
```javascript
const options = {
  url: 'https://api.example.com/contacts',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${bundle.authData.access_token}`
  },
  body: {
    email: bundle.inputData.email,
    first_name: bundle.inputData.first_name,
    last_name: bundle.inputData.last_name,
    phone: bundle.inputData.phone || null,
    company: bundle.inputData.company || null,
    tags: bundle.inputData.tags || []
  }
};

return z.request(options).then(response => {
  if (response.status !== 201 && response.status !== 200) {
    throw new z.errors.Error(`Failed to create contact: ${response.data.message}`);
  }
  return response.data;
});
```

---

## Search Actions

Find existing records in your app.

### Settings Configuration
- **Key**: `find_contact` (use "find" not "get" or "search")
- **Name**: "Find Contact"
- **Noun**: "Contact"
- **Description**: "Finds a contact by email address."

### Input Designer
Minimal fields—typically just the search criteria:
```
- email (String, Required) - "Email address to search for"
```

Or with filters:
```
- query (String, Required) - "Search term (name or email)"
- status (Dropdown) - "Filter by status" [Active, Inactive, All]
```

### API Configuration
```
Method: GET
URL: https://api.example.com/contacts?email={{bundle.inputData.email}}
```

Or with query parameters:
```
URL: https://api.example.com/contacts
Params:
  q: {{bundle.inputData.query}}
  status: {{bundle.inputData.status}}
  limit: 1
```

### Expected Response
**Must return array** (even for single result):
```json
[
  {
    "id": "contact_123",
    "email": "user@example.com",
    "first_name": "Jane"
  }
]
```

**No results = empty array** (never 404 error):
```json
[]
```

### Code Mode - Handle 404 as Empty Result
```javascript
const options = {
  url: `https://api.example.com/contacts`,
  method: 'GET',
  params: {
    email: bundle.inputData.email
  }
};

return z.request(options).then(response => {
  // Some APIs return 404 for no results - convert to empty array
  if (response.status === 404) {
    return [];
  }
  
  // Ensure we return an array
  const results = response.data;
  if (!Array.isArray(results)) {
    return results ? [results] : [];
  }
  return results;
});
```

---

## Update Actions

Modify existing records.

### Settings Configuration
- **Key**: `update_contact`
- **Name**: "Update Contact"
- **Noun**: "Contact"
- **Description**: "Updates an existing contact."

### Input Designer
**ID field is required** (use dynamic dropdown):
```
- contact_id (String, Required, Dynamic Dropdown) - "Select the contact to update"
- email (String) - "New email address"
- first_name (String) - "New first name"
- status (Dropdown) - "New status"
```

### Dynamic Dropdown for ID Selection
1. Create hidden trigger to list contacts:
   ```
   Key: list_contacts_dropdown
   Visibility: Hidden
   URL: GET https://api.example.com/contacts?limit=100
   ```

2. In Update action's Input Designer:
   - Add `contact_id` field
   - Enable "Dynamic Dropdown"
   - Select `list_contacts_dropdown` trigger
   - Value: `id`
   - Label: `email` or `name`

### API Configuration
```
Method: PATCH (or PUT)
URL: https://api.example.com/contacts/{{bundle.inputData.contact_id}}

Request Body:
{
  "email": "{{bundle.inputData.email}}",
  "first_name": "{{bundle.inputData.first_name}}",
  "status": "{{bundle.inputData.status}}"
}
```

### Code Mode - Only Send Changed Fields
```javascript
// Build body with only provided fields
const body = {};
if (bundle.inputData.email) body.email = bundle.inputData.email;
if (bundle.inputData.first_name) body.first_name = bundle.inputData.first_name;
if (bundle.inputData.status) body.status = bundle.inputData.status;

const options = {
  url: `https://api.example.com/contacts/${bundle.inputData.contact_id}`,
  method: 'PATCH',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${bundle.authData.access_token}`
  },
  body: body
};

return z.request(options).then(response => response.data);
```

---

## Search-or-Create Pattern

Combine Search and Create into one Zap step.

### Setup

1. **Create the Create action first** (e.g., "Create Contact")

2. **Create the Search action** (e.g., "Find Contact")

3. **In Search action Settings**:
   - Check "Pair an existing search and a create"
   - Select "Create Contact" from dropdown
   - Add label like "Find or Create Contact"

### User Experience
When users select this Search action:
1. Search fields appear first
2. Checkbox: "Create contact if none found"
3. If checked, Create action fields appear below
4. Zap searches first, creates only if no results

---

## Handling Line Items (Arrays)

When actions need to accept or return multiple items.

### Input: Accept Multiple Values
Set "Allows Multiples" on input field:
```
- emails (String, Allows Multiples) - "Email addresses to add"
```

User can add multiple values; Zapier sends as array:
```json
{"emails": ["a@test.com", "b@test.com", "c@test.com"]}
```

### Output: Return Multiple Results
For Search actions that might return multiple matches:

```javascript
return z.request(options).then(response => {
  // Return all results as array
  // Zapier shows first result by default
  // Users can enable "line items" to use all
  return response.data.results;
});
```

---

## Best Practices

### Field Naming
- Use same names as your app's UI
- Be specific: "Project Name" not "Name"
- Include units where relevant: "Amount (USD)"

### Help Text
- Explain format expected: "Date in YYYY-MM-DD format"
- Note defaults: "Defaults to your account's timezone if not specified"
- Mention where to find IDs: "Find this in Settings → API"

### Error Messages
```javascript
if (response.status === 400) {
  const errors = response.data.errors;
  throw new z.errors.Error(
    `Validation failed: ${errors.map(e => e.message).join(', ')}`
  );
}
```

### Deduplication
For triggers that might return same item twice:
- Always include unique `id` field in response
- Zapier uses `id` to skip duplicates automatically
