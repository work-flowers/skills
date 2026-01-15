---
name: zapier-platform-ui
description: "Guide for building private Zapier integrations using the Platform UI (Visual Builder). Use when creating custom Zapier apps with authentication (API Key, OAuth, Session Auth), polling/webhook triggers, and create/search/update actions. Covers API configuration, input field design, Code Mode for custom JavaScript, testing workflows, and integration management. Ideal for connecting internal tools or third-party APIs to Zapier without using the CLI."
---

# Zapier Platform UI Integration Builder

Build private Zapier integrations using the Platform UI at https://developer.zapier.com/

## Quick Reference

| Component | Purpose | Key Decisions |
|-----------|---------|---------------|
| Authentication | Connect user accounts | OAuth v2 (preferred) > API Key > Session > Basic |
| Triggers | Start Zaps on events | REST Hook (instant) vs Polling (1-15 min intervals) |
| Actions | Create/Search/Update data | Create (POST), Search (GET), Update (PUT/PATCH) |

## Integration Setup

1. Go to https://developer.zapier.com/ → **Start a Zapier Integration**
2. Configure: Name, Description (140 chars), Homepage URL, Logo (256x256 PNG), Intended Audience (Private/Public)
3. Set **Intended Audience** to "Private" for internal integrations

## Authentication

Choose based on your API's requirements. See `references/authentication.md` for detailed setup of each type.

**Selection guide:**
- **OAuth v2**: Best UX—users authenticate via popup, no manual credential entry
- **API Key**: User copies key from your app settings; must be self-service (no email requests)
- **Session Auth**: Exchange credentials for session token; good for APIs with token refresh
- **Basic Auth**: Least preferred; credentials entered directly into Zapier

**Test Request**: Every auth scheme needs a test endpoint (commonly `/me` or `/users/me`) that returns user data to verify credentials work.

**Connection Label**: Use `{{bundle.authData.fieldKey}}` to display account identifier in Zapier UI.

## Triggers

Triggers start Zaps when data changes. Configure in: Settings → Input Designer → API Configuration.

### Polling Triggers (Default)
Zapier polls your endpoint every 1-15 minutes (based on user's plan).

**Requirements:**
- Return array sorted reverse-chronologically (newest first)
- Each item needs unique `id` field for deduplication
- GET request to list endpoint

```
Endpoint: GET /api/items?sort=created_at&order=desc
Response: [{"id": "123", "name": "Item", "created_at": "..."}, ...]
```

### REST Hook Triggers (Instant)
Your app notifies Zapier immediately when events occur.

**Requirements:**
- Subscribe endpoint: Zapier sends `target_url` for your app to store
- Unsubscribe endpoint: Remove subscription when Zap turns off
- Perform List: GET endpoint returning same schema as webhook payload (for testing)

**Subscribe request:**
```json
POST /webhooks
{"target_url": "{{bundle.targetUrl}}", "event": "item.created"}
```

**Webhook payload to Zapier:**
```json
POST {{bundle.targetUrl}}
{"id": "123", "name": "New Item", ...}
```

## Actions

Actions let Zaps create, find, or update data in your app. See `references/actions.md` for detailed patterns.

### Create Action
- Adds new records (POST request)
- Return the created object with all fields
- Never return just `{"success": true}`

### Search Action  
- Finds existing records (GET request)
- Return array (empty array if no results—never 404)
- Only first result used by default; users can enable line items

### Update Action
- Modifies existing records (PUT/PATCH request)
- Separate from Create—don't combine into "Create or Update"
- Use Search + Update pattern instead

### Search-or-Create Pattern
Pair a Search with a Create action to "find or create" in one Zap step:
1. Build Create action first
2. In Search action settings, check "Pair an existing search and a create"
3. Select the Create action to embed

## Input Fields

Design user input forms in the **Input Designer** tab.

**Field types:**
- **String**: Text input (default)
- **Text**: Multi-line text area
- **Integer/Number**: Numeric input
- **Boolean**: Checkbox
- **DateTime**: Date picker
- **File**: File upload
- **Password**: Masked input

**Best practices:**
- Mark truly required fields as Required
- Add helpful descriptions for each field
- Use same field names as your app's UI
- Order: required fields first, related fields together

### Dynamic Dropdowns
Populate dropdowns from your API (e.g., list of projects, users, folders):

1. Create hidden trigger that fetches dropdown data
2. In action's Input Designer, add field with "Dynamic Dropdown"
3. Select the hidden trigger as data source
4. Set `value` field (ID sent to API) and `label` field (displayed to user)

## Code Mode

Switch from Form Mode when you need custom JavaScript for API calls.

**When to use:**
- Transform response data format
- Add conditional logic
- Handle non-standard authentication
- Parse non-JSON responses

**Basic structure:**
```javascript
const options = {
  url: 'https://api.example.com/endpoint',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${bundle.authData.access_token}`
  },
  body: {
    name: bundle.inputData.name,
    email: bundle.inputData.email
  }
};

return z.request(options).then(response => {
  // Transform response if needed
  return response.data;
});
```

**Available in Code Mode:**
- `z.request()` - Make HTTP requests
- `z.JSON.parse()` - Parse JSON safely
- `z.console.log()` - Debug logging
- `z.errors` - Error handling classes
- `bundle.authData` - User's auth credentials
- `bundle.inputData` - User's input field values
- `bundle.targetUrl` - Webhook URL (for REST Hook triggers)

**Error handling:**
```javascript
if (response.status === 401) {
  throw new z.errors.RefreshAuthError(); // Triggers re-auth
}
if (response.status === 400) {
  throw new z.errors.Error('Invalid request: ' + response.data.message);
}
```

**30-second timeout**: Keep code efficient; complex operations will time out.

## Testing

### In Platform UI
1. Connect test account via **Test Authentication**
2. Add test data in **Configure Test Data** 
3. Click **Test Your Request**
4. Check Response and HTTP tabs for results

### In Zap Editor
1. Create new Zap using your integration
2. Test each trigger/action with real data
3. Verify field mappings work correctly
4. For REST Hooks: Turn Zap on, trigger event in your app, verify Zap runs

## Version Management

**Private integrations (<5 users):** Edit directly in the current version.

**Private integrations (5+ users):** Create new version before editing to avoid breaking existing Zaps.

**Version workflow:**
1. Manage → Versions → Create New Version
2. Make changes in new version
3. Promote version when ready
4. Users' Zaps auto-migrate to new version

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| 401/403 | Bad credentials | Re-test authentication; check token expiry |
| 404 | Wrong endpoint URL | Verify API documentation; check URL parameters |
| 400 | Malformed request | Check request body format; verify required fields |
| Error Parsing Response | Non-JSON response | Use Code Mode to parse response |
| Task Timed Out | Response >30s or too much data | Use pagination; return less data |

## Checklist Before Sharing

- [ ] Authentication tested with valid credentials
- [ ] All triggers return data sorted newest-first
- [ ] All triggers have unique `id` field for deduplication
- [ ] Actions return created/updated object (not just success message)
- [ ] Search actions return empty array (not error) when no results
- [ ] Input fields have clear descriptions
- [ ] Tested complete Zap workflow in Zap editor
