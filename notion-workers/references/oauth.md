# OAuth Reference

Full OAuth setup flow for Notion Workers. Use when the external API requires
user-level authorisation rather than a static API key.

---

## Table of Contents

1. [When to Use OAuth vs. Secrets](#when-to-use-oauth-vs-secrets)
2. [Capability Key vs. `name`](#capability-key-vs-name)
3. [Setup Flow](#setup-flow)
4. [Configuration Options](#configuration-options)
5. [Google OAuth Example](#google-oauth-example)
6. [GitHub OAuth Example](#github-oauth-example)
7. [Salesforce OAuth Example](#salesforce-oauth-example)
8. [Generic OAuth2 Pattern](#generic-oauth2-pattern)
9. [Token Refresh and Local Testing](#token-refresh-and-local-testing)
10. [Troubleshooting](#troubleshooting)

---

## When to Use OAuth vs. Secrets

| Approach | When to Use |
|----------|------------|
| **Secrets** (`ntn workers env set`) | Service uses static API keys, service account tokens, or webhook secrets. Most third-party APIs with server-to-server auth. |
| **OAuth** (`worker.oauth()`) | Service requires user-level authorisation — the user must grant consent via a browser flow. Common for Google APIs, GitHub (when accessing user repos), Salesforce, etc. |

If the API offers both a service account/API key and OAuth, prefer the simpler
option (usually secrets) unless you specifically need per-user access scoping.

---

## Capability Key vs. `name`

`worker.oauth()` takes two identifiers and it's easy to confuse them:

```typescript
const githubAuth = worker.oauth("githubAuth", {   // ← capability key
  name: "github-oauth",                            // ← provider instance name
  // ...
});
```

- **The first argument (`"githubAuth"`) is the capability key.** This is what
  you pass to `ntn workers oauth start <key>`. It's also the JS variable name
  you use to call `accessToken()`.
- **The `name` field** is a human-readable identifier for the connected token
  on the server. Don't pass it to the CLI.

CLI commands use the capability key:

```bash
ntn workers oauth start githubAuth     # ✅ capability key
ntn workers oauth start github-oauth   # ❌ the `name` field — won't work
```

---

## Setup Flow

The full setup needs the worker deployed *before* you can store secrets and
authorise. The order matters:

### Phase 1: Define OAuth in code (no credentials yet)

```typescript
const myAuth = worker.oauth("myAuth", {
  name: "my-service-oauth",
  authorizationEndpoint: "https://provider.com/oauth/authorize",
  tokenEndpoint: "https://provider.com/oauth/token",
  scope: "read write",
  clientId: process.env.OAUTH_CLIENT_ID ?? "",
  clientSecret: process.env.OAUTH_CLIENT_SECRET ?? "",
});
```

### Phase 2: First deploy

```bash
ntn workers deploy
```

This registers the Worker with Notion. Your OAuth credentials aren't set yet —
that's fine.

### Phase 3: Configure the provider's OAuth app

Get the redirect URL Notion will use:

```bash
ntn workers oauth show-redirect-url
```

Create an OAuth app in the provider's developer settings (GitHub Developer
Settings, Google Cloud Console, Salesforce Setup, etc.), add this URL as an
authorised redirect URI, and copy the client ID and secret.

### Phase 4: Store the credentials, deploy again

```bash
ntn workers env set OAUTH_CLIENT_ID=xxx OAUTH_CLIENT_SECRET=yyy
ntn workers deploy
```

The redeploy is required so the Worker reads the new secrets.

### Phase 5: Authorise

```bash
ntn workers oauth start myAuth        # capability key, not the `name`
```

This opens a browser, you authorise, and the token is stored. The runtime
handles refresh from then on.

### Use the token in code

```typescript
worker.tool("getMyData", {
  title: "Get My Data",
  description: "Fetches data from the connected service.",
  schema: j.object({}),
  hints: { readOnlyHint: true },
  execute: async () => {
    const token = await myAuth.accessToken();
    const response = await fetch("https://api.provider.com/data", {
      headers: { Authorization: `Bearer ${token}` },
    });
    return response.json();
  },
});
```

The same `accessToken()` pattern works in syncs and webhooks.

---

## Configuration Options

| Property | Required | Description |
|----------|----------|-------------|
| `name` | Yes | Identifier for the connected token instance |
| `authorizationEndpoint` | Yes | Provider's OAuth 2.0 authorisation URL |
| `tokenEndpoint` | Yes | Provider's OAuth 2.0 token exchange URL |
| `clientId` | Yes | OAuth app's client ID |
| `clientSecret` | Yes | OAuth app's client secret |
| `scope` | Yes | Space-separated list of scopes to request |
| `authorizationParams` | No | Extra query params on the auth URL (e.g. Google's `access_type: "offline"`) |
| `accessTokenExpireMs` | No | Default token expiry in ms, for providers that don't return `expires_in` |
| `callbackUrl` | No | Override the redirect URL (rarely needed) |

---

## Google OAuth Example

Google APIs (Analytics, Sheets, Calendar, Drive, etc.) use the same OAuth2
flow with different scopes.

### 1. Create a Google Cloud OAuth app

1. Go to https://console.cloud.google.com/apis/credentials
2. Create an OAuth 2.0 Client ID (type: Web application)
3. Leave the redirect URI blank for now — you'll add it after deploying
4. Enable the relevant API (e.g., Google Analytics Data API) in the API Library
5. Note the Client ID and Client Secret

### 2. Define in code

```typescript
const googleAuth = worker.oauth("googleAuth", {
  name: "google-oauth",
  authorizationEndpoint: "https://accounts.google.com/o/oauth2/v2/auth",
  tokenEndpoint: "https://oauth2.googleapis.com/token",
  scope: "https://www.googleapis.com/auth/analytics.readonly",
  clientId: process.env.GOOGLE_CLIENT_ID ?? "",
  clientSecret: process.env.GOOGLE_CLIENT_SECRET ?? "",
  authorizationParams: {
    access_type: "offline",   // get a refresh token
    prompt: "consent",         // force consent screen, ensures refresh token is issued
  },
});
```

Common Google scopes:

| Service | Scope |
|---------|-------|
| Analytics (read) | `https://www.googleapis.com/auth/analytics.readonly` |
| Sheets (read) | `https://www.googleapis.com/auth/spreadsheets.readonly` |
| Sheets (read/write) | `https://www.googleapis.com/auth/spreadsheets` |
| Drive (read) | `https://www.googleapis.com/auth/drive.readonly` |
| Calendar (read) | `https://www.googleapis.com/auth/calendar.readonly` |
| Gmail (read) | `https://www.googleapis.com/auth/gmail.readonly` |

### 3. First deploy

```bash
ntn workers deploy
```

### 4. Configure redirect URI in Google Cloud

```bash
ntn workers oauth show-redirect-url
# Copy the output URL
```

Go back to Google Cloud Console → Credentials → your OAuth client → add the
redirect URL as an Authorised redirect URI. Save.

### 5. Store secrets and redeploy

```bash
ntn workers env set GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com GOOGLE_CLIENT_SECRET=GOCSPx-your-secret
ntn workers deploy
```

### 6. Authorise

```bash
ntn workers oauth start googleAuth
```

### Google-specific notes

- Google access tokens expire after 1 hour. The Workers runtime refreshes
  them automatically using the refresh token obtained at authorisation.
- Use `authorizationParams: { access_type: "offline", prompt: "consent" }` so
  Google issues a refresh token. Without `access_type=offline`, only a short
  access token comes back and the runtime can't refresh it.
- If the Google Cloud project is in "Testing" mode, only test users you've
  added can authorise. Switch to "Production" (and complete verification if
  needed) for broader access. Testing mode also revokes refresh tokens after
  7 days.
- For service-to-service access (no user consent needed), consider using a
  Google Service Account key stored as a secret instead of OAuth.

---

## GitHub OAuth Example

```typescript
const githubAuth = worker.oauth("githubAuth", {
  name: "github-oauth",
  authorizationEndpoint: "https://github.com/login/oauth/authorize",
  tokenEndpoint: "https://github.com/login/oauth/access_token",
  scope: "repo user",
  clientId: process.env.GITHUB_CLIENT_ID ?? "",
  clientSecret: process.env.GITHUB_CLIENT_SECRET ?? "",
});
```

1. Create a GitHub OAuth App at https://github.com/settings/developers
2. After the first deploy, run `ntn workers oauth show-redirect-url` and add
   the URL to the GitHub app
3. Store credentials: `ntn workers env set GITHUB_CLIENT_ID=... GITHUB_CLIENT_SECRET=...`

Classic GitHub OAuth app tokens don't expire by default, so refresh isn't a
concern. GitHub Apps (the newer model) issue tokens that do expire — use the
`tokenEndpoint` for whichever model your app is configured under.

---

## Salesforce OAuth Example

```typescript
const salesforceAuth = worker.oauth("salesforceAuth", {
  name: "salesforce-oauth",
  authorizationEndpoint: "https://login.salesforce.com/services/oauth2/authorize",
  tokenEndpoint: "https://login.salesforce.com/services/oauth2/token",
  scope: "api refresh_token",
  clientId: process.env.SALESFORCE_CLIENT_ID ?? "",
  clientSecret: process.env.SALESFORCE_CLIENT_SECRET ?? "",
  accessTokenExpireMs: 3600000,    // 1 hour — Salesforce doesn't return expires_in
});
```

Salesforce doesn't include `expires_in` in its token response. Set
`accessTokenExpireMs` so the runtime knows when to refresh. The `refresh_token`
scope is required to get a refresh token back.

For sandboxes, use `https://test.salesforce.com` in both endpoints.

---

## Generic OAuth2 Pattern

For any OAuth2-compliant provider:

```typescript
const serviceAuth = worker.oauth("serviceAuth", {
  name: "service-oauth",
  authorizationEndpoint: "https://provider.com/oauth/authorize",
  tokenEndpoint: "https://provider.com/oauth/token",
  scope: "read write",
  clientId: process.env.SERVICE_CLIENT_ID ?? "",
  clientSecret: process.env.SERVICE_CLIENT_SECRET ?? "",
});
```

The flow is:
1. Define `worker.oauth()` in code
2. First deploy (registers the Worker with Notion)
3. Get the redirect URL with `ntn workers oauth show-redirect-url` and configure the provider
4. Store client credentials as secrets, then deploy again
5. Run `ntn workers oauth start serviceAuth` (capability key) to authorise
6. Call `await serviceAuth.accessToken()` in your tool, sync, or webhook code

---

## Token Refresh and Local Testing

The Workers runtime handles token refresh automatically using the refresh
token obtained during the initial authorisation. The token returned by
`accessToken()` is always a valid access token at call time.

If a sync or tool starts returning auth errors after previously working, the
refresh token is likely revoked. Re-authorise:

```bash
ntn workers oauth start <capabilityKey>
```

Common reasons refresh tokens get revoked:

- Google "Testing" mode apps revoke refresh tokens after 7 days
- The user revoked the app's access in the provider's account settings
- The provider rotated app secrets
- Some providers revoke after a long period of inactivity

### Testing locally with OAuth

Once you've completed the OAuth flow on the deployed Worker, pull the
environment to get a fresh access token in your local `.env`:

```bash
ntn workers env pull
```

The server refreshes the token before returning it, so the value you pull is
always valid at pull time. Then run the capability locally:

```bash
ntn workers exec getMyData --local
```

If you get a 401 after a while, the local token has expired — run
`ntn workers env pull` again.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `invalid_client` error during OAuth | Wrong client ID or secret, or you forgot to redeploy after setting them | Verify with `ntn workers env list`, redeploy, re-run `oauth start` |
| `redirect_uri_mismatch` error | Redirect URL not configured in provider | Run `ntn workers oauth show-redirect-url`, add it to the provider's app settings exactly |
| Token works initially, then stops | Refresh token expired or revoked | Re-run `ntn workers oauth start <capabilityKey>` |
| `insufficient_scope` error | Missing required API scope | Update `scope` in `worker.oauth()`, redeploy, then re-authorise (existing token doesn't gain new scopes) |
| OAuth flow opens but fails to complete | Provider app in testing/sandbox mode | Check provider settings — may need to add test users or switch to production |
| `ntn workers oauth start` hangs | Browser didn't open or callback failed | Try copying the URL manually, or check network/firewall settings |
| `oauth start` says "capability not found" | Passed the `name` field instead of the capability key | Use the first argument from `worker.oauth("capabilityKey", ...)` |
| Google sync stops working after a few days | App in "Testing" mode → refresh token revoked after 7 days | Switch to Production, or re-authorise periodically |
| Salesforce token never refreshes | No `accessTokenExpireMs` set, runtime doesn't know when to refresh | Add `accessTokenExpireMs: 3600000` (1 hour) |
