# Authentication Reference — Zapier Platform CLI

All auth types share a common structure: a `type`, a `test` endpoint to verify credentials, optional `fields` for user input, and an optional `connectionLabel` to identify the connected account in the Zapier UI.

## Basic Auth

Zapier auto-encodes `username` and `password` as a Base64 `Authorization` header.

```js
const authentication = {
  type: 'basic',
  test: { url: 'https://example.com/api/me.json' },
  connectionLabel: '{{username}}',
  // username and password fields are provided automatically
};
```

Template: `zapier-platform init myapp --template basic-auth`

## Digest Auth

Identical setup to Basic. Zapier handles nonce/quality-of-protection automatically. Only MD5 algorithm is supported (not MD5-sess or SHA). A pre-request is made for every `z.request` to fetch the server nonce.

```js
const authentication = {
  type: 'digest',
  test: { url: 'https://example.com/api/me.json' },
  connectionLabel: (z, bundle) => bundle.inputData.username,
};
```

Template: `zapier-platform init myapp --template digest-auth`

## Custom Auth (API Key)

Most commonly used for API key authentication. You handle header injection yourself via `beforeRequest` middleware or `requestTemplate`.

```js
const authentication = {
  type: 'custom',
  test: {
    url: 'https://{{bundle.authData.subdomain}}.example.com/api/me.json',
  },
  fields: [
    { key: 'subdomain', type: 'string', required: true, helpText: 'Found in your browser address bar.' },
    { key: 'api_key', type: 'string', required: true, helpText: 'Found on your settings page.' },
  ],
};

const addApiKeyToHeader = (request, z, bundle) => {
  request.headers['X-Subdomain'] = bundle.authData.subdomain;
  const basicHash = Buffer.from(`${bundle.authData.api_key}:x`).toString('base64');
  request.headers.Authorization = `Basic ${basicHash}`;
  return request;
};

const App = {
  authentication,
  beforeRequest: [addApiKeyToHeader],
  // ...
};
```

Template: `zapier-platform init myapp --template custom-auth`

## Session Auth

Exchange user credentials (e.g. username/password) for a session key/token. Useful for APIs that issue tokens on login.

```js
const getSessionKey = async (z, bundle) => {
  const response = await z.request({
    method: 'POST',
    url: 'https://example.com/api/login.json',
    body: {
      username: bundle.authData.username,
      password: bundle.authData.password,
    },
  });
  return { sessionKey: response.data.sessionKey };
};

const authentication = {
  type: 'session',
  test: { url: 'https://example.com/api/me.json' },
  fields: [
    { key: 'username', type: 'string', required: true },
    { key: 'password', type: 'string', required: true },
  ],
  sessionConfig: { perform: getSessionKey },
};

const includeSessionKeyHeader = (request, z, bundle) => {
  if (bundle.authData.sessionKey) {
    request.headers = request.headers || {};
    request.headers['X-Session-Key'] = bundle.authData.sessionKey;
  }
  return request;
};
```

The session key is automatically stored in `bundle.authData.sessionKey` after `sessionConfig.perform` runs. For computed fields the user shouldn't see, use `{ key: 'sessionKey', type: 'string', required: false, computed: true }`.

Template: `zapier-platform init myapp --template session-auth`

## OAuth1

Implements the 3-legged OAuth1 flow (request token → authorize → access token). Matches Twitter/Trello implementations.

You must define:
- `getRequestToken` — API call to fetch the request token
- `authorizeUrl` — Where to send the user to authorise
- `getAccessToken` — Exchange request token for access token

After authentication, `bundle.authData.oauth_token` and `bundle.authData.oauth_token_secret` are available.

Set `CLIENT_ID` and `CLIENT_SECRET` (consumer key/secret) as environment variables.

The `beforeRequest` middleware should sign each request with OAuth1 credentials:

```js
const includeAccessToken = (req, z, bundle) => {
  if (bundle.authData?.oauth_token && bundle.authData?.oauth_token_secret) {
    req.auth = req.auth || {};
    _.defaults(req.auth, {
      oauth_consumer_key: process.env.CLIENT_ID,
      oauth_consumer_secret: process.env.CLIENT_SECRET,
      oauth_token: bundle.authData.oauth_token,
      oauth_token_secret: bundle.authData.oauth_token_secret,
    });
  }
  return req;
};
```

Supported signature methods: `HMAC-SHA1` (default), `HMAC-SHA256`, `RSA-SHA1`, `PLAINTEXT`.

Templates: `oauth1-trello`, `oauth1-tumblr`, `oauth1-twitter`

## OAuth2

Based on the `authorization_code` grant. The flow:
1. Zapier sends user to your `authorizeUrl`
2. User authorises and is redirected to Zapier's `redirect_uri`
3. Zapier exchanges the `code` for an `access_token`
4. Optionally refreshes the token on expiry

You must define:
- `authorizeUrl` — The authorization URL (can be object or function)
- `getAccessToken` — Exchange code for tokens

Optional:
- `refreshAccessToken` — Refresh expired tokens
- `autoRefresh: true` — Automatically refresh on 401 responses
- `scope` — OAuth scopes to request
- `enablePkce: true` — Enable PKCE extension (added v14.0.0)

```js
const authentication = {
  type: 'oauth2',
  test: { url: 'https://example.com/api/me.json' },
  oauth2Config: {
    authorizeUrl: {
      method: 'GET',
      url: 'https://example.com/oauth2/authorize',
      params: {
        client_id: '{{process.env.CLIENT_ID}}',
        state: '{{bundle.inputData.state}}',
        redirect_uri: '{{bundle.inputData.redirect_uri}}',
        response_type: 'code',
      },
    },
    getAccessToken: {
      method: 'POST',
      url: 'https://example.com/oauth2/token',
      body: {
        code: '{{bundle.inputData.code}}',
        client_id: '{{process.env.CLIENT_ID}}',
        client_secret: '{{process.env.CLIENT_SECRET}}',
        redirect_uri: '{{bundle.inputData.redirect_uri}}',
        grant_type: 'authorization_code',
      },
      headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    },
    scope: 'read,write',
  },
  fields: [
    { key: 'subdomain', type: 'string', required: true, default: 'app' },
  ],
};

const addBearerHeader = (request, z, bundle) => {
  if (bundle.authData?.access_token) {
    request.headers.Authorization = `Bearer ${bundle.authData.access_token}`;
  }
  return request;
};
```

For **PKCE**, add `enablePkce: true` to `oauth2Config` and include `code_verifier: '{{bundle.inputData.code_verifier}}'` in the `getAccessToken` body.

Template: `zapier-platform init myapp --template oauth2`

## Connection Label

Helps users identify which account is connected. Can be a template string or function:

```js
// Template string — references bundle.authData and bundle.inputData at top level
connectionLabel: '{{username}}'

// Function — use fully qualified paths
connectionLabel: (z, bundle) => bundle.inputData.username
```

Do not include sensitive data (passwords, API keys) in the connection label.

## Subdomain Validation (Security)

When your auth uses a subdomain field, validate it to prevent URL manipulation attacks:

```js
if (!/^[a-z0-9-]+$/.test(bundle.authData.subdomain)) {
  throw new Error('Subdomain can only contain letters, numbers and dashes (-).');
}
```

Apply this validation in `getAccessToken`, `refreshAccessToken`, and any other function that uses the subdomain in a URL.
