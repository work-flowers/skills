# TypeScript SDK Reference

Full API reference for `@zapier/zapier-sdk` v0.40.1.

> The same SDK surface is also available **inside a Code by Zapier step** — toggle `@zapier/zapier-sdk (latest)` in the Packages panel of the Code editor. Connections are wired through the UI rather than `findFirstConnection`, and surface as a runtime `connections["..."]` map. See the **Inside Code by Zapier** section in `SKILL.md` for the in-Zap workflow.

## Table of Contents

1. [Initialisation](#initialisation)
2. [Accounts](#accounts)
3. [Apps](#apps)
4. [Actions](#actions)
5. [Connections](#connections)
6. [HTTP Requests (fetch)](#http-requests)
7. [Tables](#tables)
8. [Client Credentials](#client-credentials)
9. [Pagination](#pagination)
10. [App Proxy Pattern](#app-proxy-pattern)
11. [Triggers (Experimental, closed beta)](#triggers-experimental)
12. [Parameter Naming (v0.40.1)](#parameter-naming-v0401)

---

## Initialisation

```typescript
import { createZapierSdk } from "@zapier/zapier-sdk";

// Browser login (uses stored credentials from `npx zapier-sdk login`)
const zapier = createZapierSdk();

// Client credentials (for servers/CI)
const zapier = createZapierSdk({
  credentials: {
    clientId: process.env.ZAPIER_CLIENT_ID,
    clientSecret: process.env.ZAPIER_CLIENT_SECRET,
  },
});

// Direct token (partner OAuth / internal use)
const zapier = createZapierSdk({
  credentials: "your-zapier-token",
});
```

The top-level `clientId` / `clientSecret` / `token` props from earlier docs are deprecated — wrap them in `credentials` in all new code.

---

## Accounts

### getProfile(options?)

Returns user profile information.

```typescript
const { data: profile } = await zapier.getProfile();
```

---

## Apps

### listApps(options?)

List available apps with optional filtering.

```typescript
const { data: apps } = await zapier.listApps({
  search: "google sheets",  // Search by name
  apps: ["slack", "gmail"], // Filter to specific apps by slug
  pageSize: 20,
  maxItems: 50,
  cursor: "...",
});
```

### getApp({ app })

Get details about a specific app. Accepts slug (`slack`), implementation name (`SlackCLIAPI`), or versioned ID (`slack@1.2.3`).

```typescript
const { data: app } = await zapier.getApp({ app: "slack" });
```

---

## Actions

### listActions({ app, actionType?, ... })

List available actions for an app.

```typescript
const { data: actions } = await zapier.listActions({
  app: "slack",
  actionType: "write",  // "read", "write", or "search"
});
```

### getAction({ app, actionType, action })

Get details about a specific action.

```typescript
const { data: action } = await zapier.getAction({
  app: "slack",
  actionType: "write",
  action: "channel_message",
});
```

### runAction({ app, actionType, action, connection?, inputs?, timeoutMs?, ... })

Execute an action and return results.

```typescript
const { data: result } = await zapier.runAction({
  app: "slack",
  actionType: "write",
  action: "channel_message",
  connection: slackConnectionId,
  inputs: {
    channel: "C0123ABCD",
    text: "Hello!",
  },
  timeoutMs: 180000, // default
});
```

### getActionInputFieldsSchema({ app, actionType, action, connection?, inputs? })

Returns JSON Schema describing input parameters. Useful for agent tool definitions.

```typescript
const { data: schema } = await zapier.getActionInputFieldsSchema({
  app: "slack",
  actionType: "write",
  action: "channel_message",
  connection: slackConnectionId,
});
```

### listActionInputFields({ app, actionType, action, connection?, inputs?, ... })

Get required input fields. Pass existing `inputs` to reveal dynamic fields.

```typescript
const { data: fields } = await zapier.listActionInputFields({
  app: "google-sheets",
  actionType: "write",
  action: "add_row",
  connection: sheetsConnectionId,
  inputs: { spreadsheet: "abc123", worksheet: "0" },
});
```

### listActionInputFieldChoices({ app, actionType, action, inputField, connection?, inputs?, ... })

Fetch dynamic dropdown options for a specific field.

```typescript
const { data: choices } = await zapier.listActionInputFieldChoices({
  app: "slack",
  actionType: "write",
  action: "channel_message",
  inputField: "channel",
  connection: slackConnectionId,
});
```

---

## Connections

### listConnections(options?)

List available connections with filtering. Non-expired connections are returned by default.

```typescript
const { data: connections } = await zapier.listConnections({
  app: "slack",
  owner: "me",           // "me" or specific owner
  includeShared: true,
  search: "production",
  title: "My Slack",
  pageSize: 20,
  // expired: true,      // pass to filter to expired-only
});
```

### findFirstConnection(options)

Returns the first matching connection. Throws if none found.

```typescript
const { data: conn } = await zapier.findFirstConnection({
  app: "slack",
  owner: "me",
});
```

### findUniqueConnection(options)

Returns exactly one matching connection. Throws if zero or multiple found.

```typescript
const { data: conn } = await zapier.findUniqueConnection({
  app: "slack",
  title: "Production Slack",
});
```

### getConnection({ connection })

Get details about a specific connection by ID or alias.

```typescript
const { data: conn } = await zapier.getConnection({ connection: 12345 });
```

---

## HTTP Requests

### fetch(url, init?)

Make authenticated HTTP requests to any API through Zapier's infrastructure. Mirrors the native `fetch(url, init?)` signature with Zapier-specific options. The `connection` option automatically injects stored credentials.

```typescript
const response = await zapier.fetch("https://slack.com/api/users.list", {
  method: "GET",
  connection: slackConnectionId,
});

// POST with body
const response = await zapier.fetch("https://api.example.com/data", {
  method: "POST",
  connection: connectionId,
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ key: "value" }),
});
```

Zapier-specific `init` options:

| Option | Description |
|--------|-------------|
| `connection` | Connection alias or ID. Strings matching a key in the `connections` map (Code-step context) are resolved against it; otherwise the value is used as a connection ID directly. |
| `callbackUrl` | URL to POST the async response to. Makes the request async. Useful for long-running calls. |
| `maxTime` | Max seconds to wait for a response. Honoured on a best-effort basis; the server may silently enforce a lower ceiling. |

**Important:** `fetch` calls are NOT governed by org-level app restrictions (unlike pre-built actions). Direct API governance is on the roadmap. For gated actions, see the approval flow in `SKILL.md`.

---

## Tables

See `tables.md` for the full Tables reference. Quick summary:

```typescript
// Create a table
const { data: table } = await zapier.createTable({ name: "Leads", description: "Sales leads" });

// Add fields
await zapier.createTableFields({ table: table.id, fields: [...] });

// Create records (max 100 per call)
await zapier.createTableRecords({ table: table.id, records: [...] });

// Query records
const { data: records } = await zapier.listTableRecords({
  table: table.id,
  filters: [
    { fieldKey: "Status", operator: "equals", value: "active" },
  ],
  sort: { fieldKey: "created_at", direction: "desc" },
});
```

---

## Client Credentials

For server/CI environments where browser login isn't possible.

```typescript
// Generate credentials (do this once via CLI)
// npx zapier-sdk create-client-credentials "my-server"
// Save the client_secret immediately — shown only once

// Use in code
const zapier = createZapierSdk({
  credentials: {
    clientId: process.env.ZAPIER_CLIENT_ID,
    clientSecret: process.env.ZAPIER_CLIENT_SECRET,
  },
});
```

SDK methods for managing credentials:

```typescript
await zapier.createClientCredentials({ name: "my-app", allowedScopes: [...] });
await zapier.listClientCredentials();
await zapier.deleteClientCredentials({ clientId: "..." });
```

---

## Pagination

Methods returning paginated results support three patterns:

```typescript
// 1. Single page
const { data: items, nextCursor } = await zapier.listActions({ app: "slack" });

// 2. Iterate over pages
for await (const page of zapier.listActions({ app: "slack" })) {
  console.log(page);
}

// 3. Iterate over individual items
for await (const item of zapier.listActions({ app: "slack" }).items()) {
  console.log(item);
}
```

Pagination parameters: `pageSize` (items per page), `maxItems` (total limit), `cursor` (resume from specific point).

---

## App Proxy Pattern

Bind a connection to an app once, then call actions with cleaner syntax:

```typescript
const slack = zapier.apps.slack({ connection: conn.id });

// These are equivalent:
await slack.write.channel_message({ inputs: { channel: "C0123", text: "Hi" } });
// vs
await zapier.runAction({
  app: "slack", actionType: "write", action: "channel_message",
  connection: conn.id, inputs: { channel: "C0123", text: "Hi" },
});
```

The proxy pattern is cleaner for multiple calls to the same app. Use `runAction` when you need more control or are working with a single call.

---

## Triggers (Experimental)

> ⚠️ **Closed beta.** Import from `"@zapier/zapier-sdk/experimental"`. Methods and behaviour may change. [Request access here](https://npsup.zapier.app/contact-us?product=Zapier%20SDK).

Subscribe to real-time events from connected apps in code — no polling, no custom webhook infrastructure.

```typescript
import { createZapierSdk } from "@zapier/zapier-sdk";
import {} from "@zapier/zapier-sdk/experimental";  // augments the SDK with trigger methods

const zapier = createZapierSdk();

// Idempotent create-or-get by name (recommended for production)
const { data: inbox } = await zapier.ensureTriggerInbox({
  name: "new-github-issues",
  app: "github",
  action: "new_issue",
  connection: ghConnId,
  // notificationUrl: "https://your-app/webhook",   // optional async push
});

// Continuously consume messages with backoff polling
await zapier.watchTriggerInbox({
  inbox: inbox.id,
  onMessage: async (msg) => {
    console.log("got issue:", msg.payload);
  },
  concurrency: 5,
  releaseOnError: true,
});
```

Method index:

| Method | Purpose |
|--------|---------|
| `createTriggerInbox` | Always create a new inbox (auto-generated name) |
| `ensureTriggerInbox` | Idempotent get-or-create by name |
| `listTriggerInboxes` / `getTriggerInbox` | Discovery |
| `pauseTriggerInbox` / `resumeTriggerInbox` | Lifecycle |
| `updateTriggerInbox` / `deleteTriggerInbox` | Mutate / tear down |
| `leaseTriggerInboxMessages` | Lease up to N messages with a lease ID |
| `ackTriggerInboxMessages` | Acknowledge (remove from inbox) |
| `releaseTriggerInboxMessages` | Release back to available pool |
| `drainTriggerInbox` | One-shot: process all currently-available messages |
| `watchTriggerInbox` | Long-running: drain + poll with backoff |
| `getTriggerInputFieldsSchema` / `listTriggerInputFields` / `listTriggerInputFieldChoices` | Trigger input discovery |

Useful signals: `ZapierReleaseTriggerMessageSignal` (throw inside `onMessage` to release without acking), `ZapierAbortDrainSignal` (throw to stop the drain after the current batch). Pass an `AbortSignal` via `signal` for clean shutdown.

---

## Parameter Naming (v0.40.1)

Parameters now use suffix-less names. Old names are deprecated but still work:

| New | Deprecated |
|-----|-----------|
| `app` | `appKey` |
| `connection` | `connectionId` |
| `action` | `actionKey` |
| `table` | `tableId` |
| `record` | `recordId` |

Use the new names in all new code.
