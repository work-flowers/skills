# Code Mode Reference

Advanced JavaScript patterns for customizing API calls in Zapier Platform UI.

## When to Use Code Mode

Switch from Form Mode when you need to:
- Transform response data into Zapier-expected format
- Add conditional logic to requests
- Handle non-JSON responses (XML, CSV, etc.)
- Make multiple sequential API calls
- Implement custom error handling
- Access Node.js standard library

## Basic Request Pattern

```javascript
const options = {
  url: 'https://api.example.com/endpoint',
  method: 'GET', // GET, POST, PUT, PATCH, DELETE
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${bundle.authData.access_token}`
  },
  params: {
    // Query parameters
    limit: 10,
    page: bundle.inputData.page || 1
  },
  body: {
    // Request body (for POST/PUT/PATCH)
    name: bundle.inputData.name,
    email: bundle.inputData.email
  }
};

return z.request(options).then(response => {
  return response.data;
});
```

## Bundle Object Reference

### bundle.authData
User's stored authentication credentials:
```javascript
bundle.authData.access_token  // OAuth token
bundle.authData.api_key       // API key
bundle.authData.username      // Basic auth username
bundle.authData.sessionToken  // Session auth token
```

### bundle.inputData
User's input from the Zap configuration:
```javascript
bundle.inputData.email        // Text field
bundle.inputData.tags         // Array if "Allows Multiples"
bundle.inputData.project_id   // Dynamic dropdown selection
```

### bundle.targetUrl
For REST Hook triggers—the unique webhook URL for this Zap:
```javascript
// In Subscribe request
body: {
  callback_url: bundle.targetUrl,
  events: ['contact.created']
}
```

### bundle.subscribeData
Data stored from Subscribe response (REST Hook triggers):
```javascript
// In Unsubscribe request
url: `https://api.example.com/webhooks/${bundle.subscribeData.webhook_id}`
```

## Response Handling

### Standard JSON Response
```javascript
return z.request(options).then(response => {
  // response.data is already parsed JSON
  return response.data;
});
```

### Transform Response Structure
```javascript
return z.request(options).then(response => {
  // API returns {results: [...], meta: {...}}
  // Zapier expects array for triggers
  return response.data.results;
});
```

### Handle Nested Data
```javascript
return z.request(options).then(response => {
  // Flatten nested structure
  const item = response.data;
  return {
    id: item.id,
    name: item.attributes.name,
    email: item.attributes.contact.email,
    created_at: item.meta.timestamps.created
  };
});
```

### Parse Non-JSON Response
```javascript
return z.request(options).then(response => {
  // For XML or other formats
  const text = response.content;
  
  // Simple XML parsing (for complex XML, use proper parser)
  const match = text.match(/<id>(\d+)<\/id>/);
  return {
    id: match ? match[1] : null,
    raw: text
  };
});
```

## Error Handling

### Error Classes

```javascript
// Generic error - shown to user
throw new z.errors.Error('Something went wrong: ' + message);

// Triggers re-authentication
throw new z.errors.RefreshAuthError();

// Halt with specific message
throw new z.errors.HaltedError('Cannot proceed: missing required data');

// Expire authentication (user must reconnect)
throw new z.errors.ExpiredAuthError();
```

### HTTP Status Code Handling
```javascript
return z.request(options).then(response => {
  if (response.status === 401) {
    throw new z.errors.RefreshAuthError();
  }
  
  if (response.status === 403) {
    throw new z.errors.Error('Access denied. Check your permissions.');
  }
  
  if (response.status === 404) {
    // For searches, return empty array instead of error
    return [];
  }
  
  if (response.status === 429) {
    throw new z.errors.Error('Rate limit exceeded. Please try again later.');
  }
  
  if (response.status >= 400) {
    const message = response.data?.message || response.data?.error || 'Unknown error';
    throw new z.errors.Error(`API Error (${response.status}): ${message}`);
  }
  
  return response.data;
});
```

### Validation Before Request
```javascript
// Validate input before making API call
if (!bundle.inputData.email || !bundle.inputData.email.includes('@')) {
  throw new z.errors.Error('Please provide a valid email address.');
}
```

## Multiple API Calls

### Sequential Requests
```javascript
// First, get the project
const projectOptions = {
  url: `https://api.example.com/projects/${bundle.inputData.project_id}`,
  method: 'GET'
};

return z.request(projectOptions).then(projectResponse => {
  const project = projectResponse.data;
  
  // Then create task in that project
  const taskOptions = {
    url: 'https://api.example.com/tasks',
    method: 'POST',
    body: {
      project_id: project.id,
      workspace_id: project.workspace_id,
      name: bundle.inputData.task_name
    }
  };
  
  return z.request(taskOptions);
}).then(taskResponse => {
  return taskResponse.data;
});
```

### Parallel Requests (Use Sparingly)
```javascript
const [usersResponse, projectsResponse] = await Promise.all([
  z.request({url: 'https://api.example.com/users'}),
  z.request({url: 'https://api.example.com/projects'})
]);

return {
  users: usersResponse.data,
  projects: projectsResponse.data
};
```

## Pagination

### Fetch All Pages (for dropdowns/triggers)
```javascript
async function fetchAllItems() {
  let allItems = [];
  let page = 1;
  let hasMore = true;
  
  while (hasMore && page <= 10) { // Safety limit
    const response = await z.request({
      url: 'https://api.example.com/items',
      params: { page: page, per_page: 100 }
    });
    
    allItems = allItems.concat(response.data.items);
    hasMore = response.data.has_more;
    page++;
  }
  
  return allItems;
}

return fetchAllItems();
```

### Cursor-Based Pagination
```javascript
async function fetchWithCursor() {
  let allItems = [];
  let cursor = null;
  
  do {
    const params = { limit: 100 };
    if (cursor) params.cursor = cursor;
    
    const response = await z.request({
      url: 'https://api.example.com/items',
      params: params
    });
    
    allItems = allItems.concat(response.data.items);
    cursor = response.data.next_cursor;
  } while (cursor && allItems.length < 500); // Safety limit
  
  return allItems;
}

return fetchWithCursor();
```

## Importing Node Libraries

```javascript
// Available via z.require()
const crypto = z.require('crypto');
const querystring = z.require('querystring');
const url = z.require('url');

// Example: Generate HMAC signature
const signature = crypto
  .createHmac('sha256', bundle.authData.api_secret)
  .update(payload)
  .digest('hex');

// Example: Build query string
const qs = querystring.stringify({
  timestamp: Date.now(),
  signature: signature
});
```

## Debugging

```javascript
// Log to Zapier's console (visible in test results)
z.console.log('Input data:', bundle.inputData);
z.console.log('Response status:', response.status);
z.console.log('Response body:', JSON.stringify(response.data, null, 2));
```

## Common Patterns

### Add Timestamp to Requests
```javascript
const timestamp = new Date().toISOString();
body: {
  ...bundle.inputData,
  created_at: timestamp,
  updated_at: timestamp
}
```

### Handle Optional Fields
```javascript
const body = {
  email: bundle.inputData.email,
  name: bundle.inputData.name
};

// Only include if provided
if (bundle.inputData.phone) {
  body.phone = bundle.inputData.phone;
}
if (bundle.inputData.tags && bundle.inputData.tags.length > 0) {
  body.tags = bundle.inputData.tags;
}
```

### Retry on Specific Errors
```javascript
async function makeRequestWithRetry(options, retries = 3) {
  for (let i = 0; i < retries; i++) {
    try {
      const response = await z.request(options);
      if (response.status === 429 && i < retries - 1) {
        await new Promise(r => setTimeout(r, 1000 * (i + 1)));
        continue;
      }
      return response;
    } catch (error) {
      if (i === retries - 1) throw error;
    }
  }
}
```

## Performance Tips

- Keep code simple—30 second timeout applies
- Avoid fetching more data than needed
- Use pagination limits
- Don't make unnecessary API calls
- Cache computed values within the request

## NPM Modules

**Not available in Platform UI.** For complex dependencies, export to Platform CLI.

Available through z.require():
- crypto
- querystring  
- url
- path
- os
- Buffer
