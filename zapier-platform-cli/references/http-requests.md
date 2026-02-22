# Making HTTP Requests — Zapier Platform CLI

## `z.request()`

The primary way to make HTTP calls. Returns a promise that resolves to a response object.

### Manual Requests (Recommended)

```js
const perform = async (z, bundle) => {
  const response = await z.request({
    url: 'https://example.com/api/items',
    method: 'POST',                              // GET, POST, PUT, PATCH, DELETE
    headers: { 'Content-Type': 'application/json' },
    body: { name: bundle.inputData.name },       // Auto-serialised to JSON
    params: { page: 1 },                         // Query string parameters
  });
  return response.data;
};
```

No need to call `JSON.stringify()` on the body — `z.request()` handles serialisation.

### Shorthand Requests

For simple GET calls, pass a URL string directly:

```js
const response = await z.request('https://example.com/api/items');
return response.data;
```

Or as an object definition (for use in resource definitions):

```js
operation: {
  perform: {
    url: 'https://example.com/api/items',
    method: 'GET',
    params: { status: 'active' },
  },
},
```

### Important: Template Literals vs Curlies

Since CLI v17, `z.request()` no longer replaces `{{var}}` placeholders and will throw an error if found. Use JavaScript template literals instead:

```js
// Correct — template literal
url: `https://${bundle.authData.subdomain}.example.com/api/items`

// Wrong in v17+ — curlies no longer work in z.request()
url: 'https://{{bundle.authData.subdomain}}.example.com/api/items'
```

Curlies (`{{...}}`) still work in **shorthand** object definitions (non-function perform) and in the **authentication** config, where they're evaluated at build time.

## Response Object

| Property | Description |
|----------|-------------|
| `response.status` | HTTP status code (200, 404, etc.) |
| `response.data` | Parsed response body (JSON parsed automatically) |
| `response.headers` | Response headers |
| `response.content` | Raw response body as string |
| `response.request` | The original request options |
| `response.skipThrowForStatus` | Set to `true` to suppress auto-throw |

Since core v10+, `response.throwForStatus()` is called automatically before the response is returned. To handle errors yourself:

```js
const response = await z.request({
  url: 'https://example.com/api/items',
  skipThrowForStatus: true,
});

if (response.status === 404) {
  return []; // Handle gracefully
}
```

## HTTP Middleware

Middleware functions run on every request (before) or response (after) in your integration.

### `beforeRequest` — Modify Outgoing Requests

Commonly used to add authentication headers:

```js
const addAuthHeader = (request, z, bundle) => {
  request.headers['Authorization'] = `Bearer ${bundle.authData.access_token}`;
  return request;
};

const App = {
  beforeRequest: [addAuthHeader],
  // ...
};
```

### `afterResponse` — Process Incoming Responses

Commonly used to parse non-JSON responses or handle API-specific error patterns:

```js
const handleErrors = (response, z, bundle) => {
  if (response.status === 200 && response.data?.success === false) {
    throw new z.errors.Error(response.data.message, response.data.code);
  }
  return response;
};

const parseXML = (response, z, bundle) => {
  // For APIs that return XML
  response.data = xml.parse(response.content);
  return response;
};

const App = {
  afterResponse: [parseXML, handleErrors],
  // ...
};
```

Your `afterResponse` middleware runs before the built-in `throwForStatus`, so you can set `response.skipThrowForStatus = true` to prevent auto-throwing for specific status codes.

## Request Template

Set default options for all `z.request()` calls:

```js
const App = {
  requestTemplate: {
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
    },
  },
  // ...
};
```

## Raw Requests (File Downloads)

Set `raw: true` to get the response as a stream (useful for file downloads):

```js
const fileRequest = z.request({
  url: 'https://example.com/file.pdf',
  raw: true,
});
const stashedUrl = await z.stashFile(fileRequest);
```

## Encoding

`z.request()` auto-encodes non-ASCII characters and reserved characters in URLs. Use `skipEncodingChars` to customise which characters are not encoded.

## Error Handling Summary

| Error Class | When to Use | Effect |
|-------------|-------------|--------|
| `z.errors.Error(msg, code, status)` | User-facing errors (bad input, API errors) | Zap turns off if too frequent |
| `z.errors.HaltedError(msg)` | Soft failure (e.g. duplicate record) | Zap stays on, task not replayable |
| `z.errors.ExpiredAuthError(msg)` | Credentials expired | User prompted to re-auth |
| `z.errors.RefreshAuthError()` | Token needs refresh | Auto-refreshes OAuth2/Session tokens |
| `z.errors.ThrottledError(msg, seconds)` | Rate limited (429) | Retried after specified delay |

Best practice: elaborate on terse API error messages for users. "not_authenticated" → "Your API Key is invalid. Please reconnect your account."
