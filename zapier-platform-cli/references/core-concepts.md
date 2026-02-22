# Core Concepts — Zapier Platform CLI

## The `z` Object

Provided as the first argument to all perform functions. Contains Zapier-specific utilities.

### Key Methods

| Method | Purpose |
|--------|---------|
| `z.request(url, options)` | HTTP client (see `references/http-requests.md`) |
| `z.console.log(...)` | Log to Zapier's console (view via `zapier-platform logs --type=console`) |
| `z.JSON.parse(string)` | Safe JSON parsing |
| `z.dehydrate(func, inputData, cacheExpiration?)` | Create a lazy-loading pointer to data |
| `z.dehydrateFile(func, inputData, cacheExpiration?)` | Create a lazy-loading pointer to a file |
| `z.stashFile(content, length?, filename?, contentType?)` | Upload file content to Zapier's file store |
| `z.cursor.get()` | Retrieve stored cursor for pagination |
| `z.cursor.set(value)` | Store cursor for next page |
| `z.errors.*` | Error classes (see HTTP requests reference) |
| `z.generateCallbackUrl()` | Generate a callback URL for async operations |

### `z.console`

Works like Node.js `Console` class. Logs are stored in Zapier's distributed datastore.

```js
z.console.log('Processing item:', item.id);
z.console.error('Something went wrong:', error.message);
```

View logs: `zapier-platform logs --type=console`

## The `bundle` Object

Provided as the second argument to all perform functions. Contains contextual data for the current execution.

### `bundle.authData`

User's authentication data. Contents depend on auth type:

| Auth Type | Available Fields |
|-----------|-----------------|
| Basic | `username`, `password` |
| Custom | Whatever keys you defined in `fields` |
| Session | Fields from `fields` + values returned by `sessionConfig.perform` |
| OAuth1 | `oauth_token`, `oauth_token_secret` + your custom fields |
| OAuth2 | `access_token`, `refresh_token` + your custom fields |

### `bundle.inputData`

User-provided values from the Zap step's input fields:

```js
// If user filled in fields: name="Jane", email="jane@example.com"
bundle.inputData.name    // "Jane"
bundle.inputData.email   // "jane@example.com"
```

### `bundle.inputDataRaw`

Like `inputData` but before Zapier processes it — curlies (`{{123__field_name}}`) are still unresolved, friendly datetimes haven't been parsed. Rarely needed.

### `bundle.meta`

Runtime metadata about the current execution:

| Property | Type | Description |
|----------|------|-------------|
| `isLoadingSample` | boolean | True during "Test Trigger" in editor |
| `isFillingDynamicDropdown` | boolean | True when loading dynamic dropdown data |
| `isPopulatingDedupe` | boolean | True on first Zap activation (backfill) |
| `isTestingAuth` | boolean | True during "Test Authentication" |
| `page` | number | Current page number (for dynamic dropdown paging, starts at 0) |
| `limit` | number | Suggested result limit (-1 if not applicable) |
| `zap.id` | string | The Zap's ID |

### `bundle.targetUrl` (REST Hook triggers only)

The unique webhook URL Zapier generated for this Zap. Send your webhook payloads here:

```js
performSubscribe: async (z, bundle) => {
  await z.request({
    method: 'POST',
    url: 'https://example.com/webhooks',
    body: { url: bundle.targetUrl, event: 'new_order' },
  });
};
```

### `bundle.subscribeData` (REST Hook triggers only)

The data returned from `performSubscribe`. Available in `perform` and `performUnsubscribe`. Typically contains the webhook subscription ID.

### `bundle.cleanedRequest` / `bundle.rawRequest` (REST Hook triggers only)

The incoming webhook payload. `cleanedRequest` is the parsed body; `rawRequest` is the full HTTP request object including headers.

### `bundle.outputData` (Callback triggers/creates only)

Contains the output from the initial `perform` before the callback URL was invoked. Useful in `performResume`.

## Dehydration

Lazily load data that's expensive to fetch upfront. Common pattern: a trigger returns 100 items but detailed data for each is only needed when a specific item triggers a Zap.

### Dehydrating Data

```js
const getMovieDetails = async (z, bundle) => {
  const response = await z.request(`https://example.com/movies/${bundle.inputData.id}`);
  return response.data;
};

const movieList = async (z, bundle) => {
  const response = await z.request('https://example.com/movies');
  return response.data.map(movie => {
    movie.details = z.dehydrate(getMovieDetails, { id: movie.id });
    return movie;
  });
};

// Don't forget to register the hydrator
const App = {
  hydrators: { getMovieDetails },
  // ...
};
```

The third argument to `z.dehydrate()` is an optional `cacheExpiration` in seconds (default 300, max 86400).

### Merging Hydrated Data

Use `$HOIST$` as the key to merge hydrated data into the parent object instead of nesting it:

```js
movie.$HOIST$ = z.dehydrate(getMovieDetails, { id: movie.id });
// After hydration, movie will have all properties from getMovieDetails at the top level
```

### File Dehydration

Use `z.dehydrateFile()` instead of `z.dehydrate()` when the data is a file. Zapier delays downloading the file until it's actually needed:

```js
pdf.file = z.dehydrateFile(downloadPDF, { url: pdf.download_url });
```

## Stashing Files

Upload files to Zapier's temporary file store. Accepts `String`, `Buffer`, or `Stream`:

```js
// From a string
const url = await z.stashFile('Hello world!', 12, 'hello.txt', 'text/plain');

// From another URL (most common)
const fileRequest = z.request({ url: 'https://example.com/file.pdf', raw: true });
const stashedUrl = await z.stashFile(fileRequest);
```

Only use `z.stashFile()` in hydration methods or hook trigger performs with short-lived URLs. Don't stash dozens of files in a polling call — it's expensive.

### Full File Example (Dehydration + Stashing)

```js
const downloadFile = (z, bundle) => {
  const filePromise = z.request({ url: bundle.inputData.downloadUrl, raw: true });
  return z.stashFile(filePromise);
};

const listFiles = async (z, bundle) => {
  const response = await z.request('https://example.com/api/files');
  return response.data.map(file => {
    file.file = z.dehydrateFile(downloadFile, { downloadUrl: file.secret_url });
    delete file.secret_url;
    return file;
  });
};

const App = {
  hydrators: { downloadFile },
  triggers: {
    new_file: {
      noun: 'File',
      display: { label: 'New File', description: 'Triggers on new files.' },
      operation: { perform: listFiles },
    },
  },
};
```

## Throttle Configuration

Rate-limit your actions to protect your API. Set at the root level (default for all actions) or per-action.

```js
const App = {
  throttle: {
    window: 600,        // 10-minute window
    limit: 50,          // Max 50 invocations per window
    scope: ['account'], // Scope: 'user', 'auth', 'account', 'action'
  },
  creates: {
    upload_video: {
      operation: {
        throttle: {
          window: 600,
          limit: 5,       // Stricter limit for this action
          retry: false,   // Don't auto-retry throttled tasks
          scope: ['account', 'action'],
        },
        perform: () => {},
      },
    },
  },
};
```

## Callback / Async Operations

For long-running operations, use `z.generateCallbackUrl()` to create a URL that your server can POST to when the operation completes:

```js
const perform = async (z, bundle) => {
  const callbackUrl = z.generateCallbackUrl();
  await z.request({
    url: 'https://example.com/api/long-task',
    method: 'POST',
    body: {
      data: bundle.inputData,
      callback_url: callbackUrl,
    },
  });
  return {}; // Return immediately
};

const performResume = async (z, bundle) => {
  // Called when your server POSTs to the callback URL
  return { ...bundle.outputData, ...bundle.cleanedRequest };
};
```

Your server has a maximum of 30 days to POST to the callback URL.
