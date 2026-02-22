---
name: zapier-platform-cli
description: "Guide for building Zapier integrations using the Platform CLI (Node.js). Use this skill when the user wants to build a Zapier integration with code, create custom triggers/searches/creates using JavaScript or TypeScript, set up authentication (Basic, Custom/API Key, Session, OAuth1, OAuth2, Digest), scaffold resources, write tests, or deploy CLI-based integrations. Also trigger when the user mentions zapier-platform-cli, zapier-platform init, zapier push, z.request, bundle object, or writing Node.js code for a Zapier app. This skill covers the full CLI development lifecycle: project setup, authentication, triggers (polling and REST hooks), searches, creates, HTTP requests, input/output fields, dehydration, file handling, error handling, environment variables, testing, deployment, and version management. Prefer this skill over the Platform UI skill whenever the user indicates they want to build with code, use npm modules, need TypeScript support, or require capabilities beyond what the visual builder offers."
---

# Zapier Platform CLI Integration Builder

Build Zapier integrations with Node.js using the CLI at https://github.com/zapier/zapier-platform

Current CLI version: **18.1.0**. Integrations run on Node.js **v22**.

## Quick Reference

| Concept | What It Does | Key Command |
|---------|-------------|-------------|
| Init | Scaffold a new project from a template | `zapier-platform init myapp --template minimal` |
| Auth | Connect user accounts | Define `authentication` in `index.js` |
| Triggers | Start Zaps on events | Polling (array, newest first) or REST Hook |
| Searches | Find existing records | Return array, best match first |
| Creates | Add new records | Return the created object |
| Resources | REST-like CRUD wrapper | Generates triggers/creates/searches automatically |
| Push | Deploy to Zapier | `zapier-platform push` |

> **Binary name update**: The CLI binary changed from `zapier` to `zapier-platform`. The old name still works but is deprecated.

## Project Setup

```bash
# Install CLI globally
npm install -g zapier-platform-cli

# Authenticate (use --sso flag for Google/Facebook/Microsoft SSO login)
zapier-platform login

# Create new project from template
zapier-platform init myapp --template minimal
cd myapp && npm install

# Register with Zapier (required before first push)
zapier-platform register "My Integration"

# Run tests, validate, and push
zapier-platform test
zapier-platform validate
zapier-platform push
```

Available templates: `minimal`, `basic-auth`, `custom-auth`, `digest-auth`, `session-auth`, `oauth1-trello`, `oauth2`, `files`, `resource`, `search`, `dynamic-dropdown`, and more. Run `zapier-platform init` to see the full list.

### Project Structure

```
myapp/
├── index.js          # Main entry point — exports the App definition
├── package.json      # Version here = your integration version
├── triggers/         # Trigger modules
├── creates/          # Create action modules
├── searches/         # Search action modules
├── resources/        # Resource modules (optional, generates triggers/creates/searches)
├── test/             # Test files
├── .env              # Local environment variables (don't commit)
└── node_modules/
```

### App Definition (`index.js`)

```js
const App = {
  version: require('./package.json').version,
  platformVersion: require('zapier-platform-core').version,

  authentication: {},       // See references/authentication.md
  hydrators: {},            // Register dehydration functions
  requestTemplate: {},      // Default request options applied to all z.request calls
  beforeRequest: [],        // Middleware: modify requests before they're sent
  afterResponse: [],        // Middleware: process responses before they're returned

  resources: {},            // REST-like resource definitions
  triggers: {},             // Trigger definitions
  searches: {},             // Search definitions
  creates: {},              // Create definitions
};

module.exports = App;
```

## Authentication

Choose the right auth method for your API. See `references/authentication.md` for full implementation patterns for each type.

| Method | When to Use | Template |
|--------|-------------|----------|
| **Basic** | Username + password | `--template basic-auth` |
| **Digest** | Like Basic but with nonce exchange | `--template digest-auth` |
| **Custom** | API keys, tokens, any non-standard auth | `--template custom-auth` |
| **Session** | Exchange credentials for a session token | `--template session-auth` |
| **OAuth1** | 3-legged OAuth (Twitter/Trello style) | `--template oauth1-trello` |
| **OAuth2** | Authorization code flow (most common for SaaS) | `--template oauth2` |

All auth types need a `test` property — an API endpoint that verifies credentials work (commonly `/me` or `/users/me`).

Set secrets as environment variables, never hardcode them:

```bash
zapier-platform env:set 1.0.0 CLIENT_ID=your_id
zapier-platform env:set 1.0.0 CLIENT_SECRET=your_secret
```

## Triggers, Searches, and Creates

These are the core building blocks of any integration. See `references/triggers-searches-creates.md` for detailed patterns and examples.

### Return Types

| Method | Must Return | Notes |
|--------|-------------|-------|
| Trigger | `Array` | 0+ objects; used by deduper if polling |
| Search | `Array` | 0+ objects; best match first |
| Create | `Object` | The created record |

### Basic Trigger Example

```js
const newRecipe = {
  key: 'new_recipe',
  noun: 'Recipe',
  display: {
    label: 'New Recipe',
    description: 'Triggers when a new recipe is added.',
  },
  operation: {
    perform: async (z, bundle) => {
      const response = await z.request('https://example.com/api/recipes');
      return response.data; // Must be an array, newest first
    },
    sample: { id: 1, name: 'Falafel', style: 'mediterranean' },
    outputFields: [
      { key: 'id', label: 'Recipe ID', type: 'integer' },
      { key: 'name', label: 'Name', type: 'string' },
    ],
  },
};
```

### Scaffolding

```bash
zapier-platform scaffold trigger "New Contact"
zapier-platform scaffold create "Add Contact"
zapier-platform scaffold search "Find Contact"
zapier-platform scaffold resource "Contact"    # Generates trigger + create + search
```

## The `z` Object and `bundle` Object

Every perform function receives `(z, bundle)`. See `references/core-concepts.md` for detailed reference.

**Key `z` methods:**
- `z.request(url, options)` — Make HTTP requests (see `references/http-requests.md`)
- `z.console.log()` — Write to Zapier's log (viewable via `zapier-platform logs --type=console`)
- `z.dehydrate(func, inputData)` — Lazy-load expensive data
- `z.dehydrateFile(func, inputData)` — Lazy-load files
- `z.stashFile(content, length, filename, contentType)` — Upload files to Zapier's file store
- `z.errors.Error(message, code, status)` — Throw user-friendly errors
- `z.errors.HaltedError(message)` — Soft-fail without turning off the Zap
- `z.errors.ExpiredAuthError(message)` — Prompt user to re-authenticate
- `z.errors.RefreshAuthError()` — Trigger OAuth2/Session token refresh
- `z.errors.ThrottledError(message, retryAfterSeconds)` — Rate-limit retry
- `z.cursor.get()` / `z.cursor.set(value)` — For cursor-based paging in dynamic dropdowns

**Key `bundle` properties:**
- `bundle.authData` — Authentication credentials (access_token, api_key, etc.)
- `bundle.inputData` — User-provided input field values for this run
- `bundle.meta` — Runtime metadata (isLoadingSample, isFillingDynamicDropdown, page, etc.)
- `bundle.targetUrl` — Webhook URL (REST Hook triggers only)
- `bundle.subscribeData` — Data from performSubscribe (REST Hook triggers only)
- `bundle.cleanedRequest` — Incoming webhook payload (REST Hook triggers only)
- `bundle.rawRequest` — Raw incoming request (REST Hook triggers only)
- `bundle.outputData` — Output from initial perform (callback triggers only)

## HTTP Requests

See `references/http-requests.md` for full details on shorthand vs manual requests, middleware, and response handling.

**Quick example:**

```js
const perform = async (z, bundle) => {
  const response = await z.request({
    url: 'https://example.com/api/items',
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: { name: bundle.inputData.name },
  });
  return response.data;
};
```

Since core v10+, `response.throwForStatus()` is called automatically. Set `skipThrowForStatus: true` on the request to handle errors yourself.

## Input Fields

Define what users fill in when configuring a Zap step:

```js
inputFields: [
  { key: 'name', label: 'Name', type: 'string', required: true, helpText: 'Contact full name' },
  { key: 'email', label: 'Email', type: 'string', required: true },
  { key: 'role', label: 'Role', type: 'string', choices: { admin: 'Admin', user: 'User' } },
  fetchDynamicFields, // Can also be async functions that return field arrays
],
```

**Field types:** `string`, `text`, `integer`, `number`, `boolean`, `datetime`, `file`, `password`, `copy`.

For dynamic dropdowns, use a trigger's key as the `dynamic` property on the field. See `references/triggers-searches-creates.md` for details.

## Environment Variables

```bash
# Set on Zapier (per-version, auto-copied to new versions)
zapier-platform env:set 1.0.0 MY_KEY=value

# View
zapier-platform env:get 1.0.0

# Local testing: use .env file
echo "MY_KEY=value" >> .env
```

Access in code via `process.env.MY_KEY`. Variables set via `env:set` are always uppercased.

In tests, call `zapier.tools.env.inject()` to load `.env` file values.

## Testing and Debugging

See `references/testing-deployment.md` for full testing patterns.

```bash
# Run tests
zapier-platform test

# View logs
zapier-platform logs --type=console
zapier-platform logs --type=http --detailed
zapier-platform logs --type=bundle
```

Write tests using any Node.js test framework (Mocha is common):

```js
const zapier = require('zapier-platform-core');
const App = require('../index');
const appTester = zapier.createAppTester(App);

describe('triggers', () => {
  zapier.tools.env.inject();

  it('should load recipes', async () => {
    const bundle = { inputData: {} };
    const results = await appTester(App.triggers.new_recipe.operation.perform, bundle);
    expect(results.length).toBeGreaterThan(0);
    expect(results[0]).toHaveProperty('id');
  });
});
```

## Deployment and Version Management

See `references/testing-deployment.md` for full deployment workflow.

```bash
zapier-platform push                         # Deploy current version
zapier-platform versions                     # List versions
zapier-platform promote 1.0.1               # Make public (requires review for first time)
zapier-platform migrate 1.0.0 1.0.1 100%    # Move users to new version
zapier-platform deprecate 1.0.0 2025-12-01  # Sunset old version
zapier-platform users:add user@example.com 1.0.0  # Share with specific user
zapier-platform team:add user@example.com    # Add admin collaborator
```

Update your version in `package.json` before pushing a new version. Environment variables are copied from the previous version automatically.

## TypeScript Support

TypeScript is a first-class language since CLI v17. Use `--language typescript` with `zapier-platform init`:

```bash
zapier-platform init myapp --template oauth2 --language typescript
```

Import types from `zapier-platform-core`:

```ts
import type { Authentication, PollingTriggerPerform, InferInputData } from 'zapier-platform-core';
```

## Common Patterns

**Middleware for adding auth headers:**
```js
const addBearerHeader = (request, z, bundle) => {
  if (bundle.authData?.access_token) {
    request.headers.Authorization = `Bearer ${bundle.authData.access_token}`;
  }
  return request;
};

const App = {
  // ...
  beforeRequest: [addBearerHeader],
};
```

**Subdomain validation (security best practice for OAuth):**
```js
if (!/^[a-z0-9-]+$/.test(bundle.authData.subdomain)) {
  throw new Error('Subdomain can only contain letters, numbers and dashes.');
}
```

**Using npm modules:**
```bash
npm install --save some-module
```
Then use normally with `require()`. Modules are re-installed fresh during `zapier-platform push`.
