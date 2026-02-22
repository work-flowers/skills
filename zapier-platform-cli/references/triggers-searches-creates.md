# Triggers, Searches, Creates, and Resources — Zapier Platform CLI

## Triggers

Triggers watch for new data and start Zaps. Two types: **polling** and **REST Hook** (instant).

### Polling Triggers

Zapier calls your perform function every 1–15 minutes. Return an array sorted newest-first. Each item needs a unique `id` field for deduplication.

```js
const newContact = {
  key: 'new_contact',
  noun: 'Contact',
  display: {
    label: 'New Contact',
    description: 'Triggers when a new contact is created.',
  },
  operation: {
    inputFields: [
      { key: 'list_id', label: 'List', type: 'string', dynamic: 'list_lists.id.name' },
    ],
    perform: async (z, bundle) => {
      const response = await z.request({
        url: 'https://example.com/api/contacts',
        params: {
          list_id: bundle.inputData.list_id,
          sort: 'created_at',
          order: 'desc',
        },
      });
      return response.data;
    },
    sample: {
      id: '123',
      name: 'Jane Doe',
      email: 'jane@example.com',
      created_at: '2025-01-15T10:30:00Z',
    },
    outputFields: [
      { key: 'id', label: 'Contact ID', type: 'string' },
      { key: 'name', label: 'Name', type: 'string' },
      { key: 'email', label: 'Email', type: 'string' },
    ],
  },
};
```

### REST Hook Triggers (Instant)

Your app sends data to Zapier immediately when events occur. You must implement three functions:

- `performSubscribe` — Register a webhook URL with your API
- `performUnsubscribe` — Remove the webhook when the Zap is turned off
- `perform` — Process inbound webhook payloads (return array)
- `performList` — Fallback poll for testing in the Zap editor (return same schema as webhook)

```js
const newOrder = {
  key: 'new_order',
  noun: 'Order',
  display: {
    label: 'New Order',
    description: 'Triggers instantly when a new order is placed.',
  },
  operation: {
    type: 'hook',
    performSubscribe: async (z, bundle) => {
      const response = await z.request({
        url: 'https://example.com/api/webhooks',
        method: 'POST',
        body: {
          url: bundle.targetUrl,
          event: 'order.created',
        },
      });
      return response.data; // Must include an id for unsubscribe
    },
    performUnsubscribe: async (z, bundle) => {
      const response = await z.request({
        url: `https://example.com/api/webhooks/${bundle.subscribeData.id}`,
        method: 'DELETE',
      });
      return response.data;
    },
    perform: async (z, bundle) => {
      // Process the incoming webhook payload
      return [bundle.cleanedRequest]; // Must return an array
    },
    performList: async (z, bundle) => {
      // Fallback for testing — fetch recent orders
      const response = await z.request('https://example.com/api/orders?limit=5');
      return response.data;
    },
    sample: { id: '456', total: 29.99, status: 'pending' },
  },
};
```

When a trigger returns an empty array, the Zap does not fire. For REST Hooks this can be used to filter data.

If webhook subscriptions expire, return an `expiration_date` (ISO 8601) from the subscribe endpoint and Zapier will auto-resubscribe.

### Hidden Triggers (for Dynamic Dropdowns)

Create a trigger that isn't visible to users but powers a dynamic dropdown:

```js
const listProjects = {
  key: 'list_projects',
  noun: 'Project',
  display: {
    label: 'List Projects',
    description: 'Hidden trigger for dynamic dropdown.',
    hidden: true,
  },
  operation: {
    perform: async (z, bundle) => {
      const response = await z.request('https://example.com/api/projects');
      return response.data;
    },
  },
};
```

Reference in an input field:
```js
{ key: 'project_id', label: 'Project', type: 'string', dynamic: 'list_projects.id.name' }
```

Format: `dynamic: '{trigger_key}.{value_field}.{label_field}'`

### Paging in Dynamic Dropdowns

When a trigger powers a dynamic dropdown, use `bundle.meta.page` for pagination and set `canPaginate: true`:

```js
operation: {
  canPaginate: true,
  perform: async (z, bundle) => {
    const response = await z.request({
      url: 'https://example.com/api/items',
      params: {
        limit: 100,
        offset: 100 * bundle.meta.page,
      },
    });
    return response.data;
  },
},
```

For cursor-based paging, use `z.cursor.get()` and `z.cursor.set()`.

## Searches

Searches find existing records. Return an array (empty if nothing found — never throw 404). Best match should be first.

```js
const findContact = {
  key: 'find_contact',
  noun: 'Contact',
  display: {
    label: 'Find Contact',
    description: 'Finds a contact by email address.',
  },
  operation: {
    inputFields: [
      { key: 'email', label: 'Email', type: 'string', required: true },
    ],
    perform: async (z, bundle) => {
      const response = await z.request({
        url: 'https://example.com/api/contacts',
        params: { email: bundle.inputData.email },
      });
      return response.data; // Array of matches
    },
    sample: { id: '123', name: 'Jane Doe', email: 'jane@example.com' },
  },
};
```

### Search-or-Create

Pair a search with a create so Zapier can "find or create" in one step:

```js
const findOrCreateContact = {
  key: 'find_or_create_contact',
  display: {
    label: 'Find or Create Contact',
    description: 'Finds or creates a contact.',
  },
  search: 'find_contact',   // key of the search
  create: 'create_contact', // key of the create
};
```

Register under `searchOrCreates` in your App definition.

## Creates

Creates add new records. Return the created object (not just `{ success: true }`).

```js
const createContact = {
  key: 'create_contact',
  noun: 'Contact',
  display: {
    label: 'Create Contact',
    description: 'Creates a new contact.',
  },
  operation: {
    inputFields: [
      { key: 'name', label: 'Name', type: 'string', required: true },
      { key: 'email', label: 'Email', type: 'string', required: true },
      { key: 'phone', label: 'Phone', type: 'string' },
    ],
    perform: async (z, bundle) => {
      const response = await z.request({
        url: 'https://example.com/api/contacts',
        method: 'POST',
        body: {
          name: bundle.inputData.name,
          email: bundle.inputData.email,
          phone: bundle.inputData.phone,
        },
      });
      return response.data;
    },
    sample: { id: '789', name: 'Jane Doe', email: 'jane@example.com' },
  },
};
```

## Resources

A resource is a REST-like wrapper that auto-generates triggers, creates, and searches from a single definition:

```js
const Contact = {
  key: 'contact',
  noun: 'Contact',
  list: {
    display: { label: 'New Contact', description: 'Triggers on new contacts.' },
    operation: {
      perform: { url: 'https://example.com/api/contacts' },
    },
  },
  create: {
    display: { label: 'Create Contact', description: 'Creates a contact.' },
    operation: {
      perform: {
        method: 'POST',
        url: 'https://example.com/api/contacts',
        body: { name: '{{bundle.inputData.name}}' },
      },
    },
  },
};
```

Auto-generated keys follow the pattern: `contactList`, `contactCreate`, `contactSearch`, `contactSearchOrCreate`.

Scaffold with: `zapier-platform scaffold resource Contact`

## Output Fields

Define labels and types for the data your trigger/search/create returns. Used in the Zap editor when users map fields.

```js
outputFields: [
  { key: 'id', label: 'Contact ID', type: 'integer' },
  { key: 'author__name', label: 'Author Name', type: 'string' },       // Nested field
  { key: 'items[]name', label: 'Item Name', type: 'string' },           // Line item field
  fetchDynamicOutputFields, // Can also be async functions
],
```

Set `primary: true` on field(s) that form the unique key for deduplication.

## Fallback Samples

Provide a `sample` object matching the schema of your perform's return data. This is shown when Zapier can't fetch live data:

```js
sample: {
  id: 1,
  title: 'Example Record',
  created_at: '2025-01-15T10:00:00Z',
},
```
