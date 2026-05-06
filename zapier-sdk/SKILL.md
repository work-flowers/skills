---
name: zapier-sdk
description: "Guide for using the Zapier SDK and CLI to explore apps, discover actions, and run authenticated API calls against 9,000+ connected apps. Trigger when the user mentions the Zapier SDK, zapier-sdk, @zapier/zapier-sdk, wants to call a connected app's API through Zapier's auth infrastructure, run a one-off action via CLI (Slack message, Google Sheet, HubSpot lookup), make authenticated fetch/curl requests through Zapier, or list connected apps. Also trigger for using the SDK inside a Code by Zapier step — the @zapier/zapier-sdk toggle, the connections[] runtime map, calling zapier.apps.X or zapier.fetch from a Code step, or collapsing a multi-branch / multi-sub-zap workflow into one Code step. This skill is for CONSUMING Zapier's pre-built connectors, NOT for BUILDING new Zapier integrations (that's zapier-platform-cli and zapier-platform-ui)."
---

# Zapier SDK — Explore Apps and Run Actions

The Zapier SDK gives you programmatic access to Zapier's full app ecosystem. Any API call, on behalf of a user, with no OAuth setup. Zapier handles auth, token refresh, retries, and API quirks across 9,000+ integrations.

**Current versions:** SDK `0.40.1` / CLI `0.39.1` (open beta, free during early access)

## When to Use This Skill vs Others

This skill is for **consuming** Zapier's pre-built connectors — calling Slack, Google Sheets, Salesforce, etc. through Zapier's auth layer using the SDK or CLI.

The **zapier-platform-cli** and **zapier-platform-ui** skills are for **building** new Zapier integrations (creating custom triggers, actions, and auth flows that get published to Zapier's app directory). Those are a completely different product.

Rule of thumb: if the user wants to *call* an app's API, use this skill. If they want to *build* an integration that appears in Zapier's app catalog, use the Platform skills.

## Prerequisites

- Node.js 20+
- `npm install @zapier/zapier-sdk` (runtime)
- `npm install -D @zapier/zapier-sdk-cli @types/node typescript` (dev/CLI)
- Logged in (see authentication section below)
- At least one connected app at https://zapier.com/app/assets/connections

## Authentication — Local vs Headless Environments

The SDK/CLI needs Zapier credentials. How you provide them depends on the environment:

**Local terminal or Claude Code (has a browser):**
```bash
npx zapier-sdk login
```
This opens a browser for OAuth. Credentials are stored at `~/.config/zapier-sdk-cli-nodejs/config.json`.

**Cowork, CI/CD, or any headless environment (no browser):**

Browser-based login won't work here. Use **client credentials** instead:

1. **Generate credentials once** from a local machine where you can open a browser:
   ```bash
   npx zapier-sdk create-client-credentials "cowork"
   ```
   Save the `client_id` and `client_secret` immediately — the secret is shown only once.

2. **Use them in Cowork** by passing the flags to any CLI command:
   ```bash
   npx zapier-sdk list-connections \
     --credentials-client-id YOUR_CLIENT_ID \
     --credentials-client-secret YOUR_SECRET \
     --owner me --json
   ```

   Or export them as environment variables to avoid repeating the flags:
   ```bash
   export ZAPIER_CLIENT_ID="your_client_id"
   export ZAPIER_CLIENT_SECRET="your_client_secret"
   npx zapier-sdk list-connections --owner me --json
   ```

For the TypeScript SDK, pass them at initialisation:
```typescript
const zapier = createZapierSdk({
  clientId: process.env.ZAPIER_CLIENT_ID,
  clientSecret: process.env.ZAPIER_CLIENT_SECRET,
});
```

## CLI Workflow — The Core Loop

The CLI is the fastest way to explore what's possible and run one-off actions. The typical workflow goes: find app → find connection → discover actions → inspect inputs → run action.

### 1. Find the App

```bash
npx zapier-sdk list-apps --search "slack" --json
```

This returns app metadata including the `slug` (e.g. `slack`, `google-sheets`). Use the slug in all subsequent commands.

### 2. Find Your Connection

```bash
npx zapier-sdk list-connections --app slack --owner me --json
```

Or to get the first non-expired match directly:

```bash
npx zapier-sdk find-first-connection --app slack --owner me --is-expired false --json
```

Each connection has a numeric `id` — you'll pass this as `--connection-id` (or `--connection`) when running actions.

### 3. Discover Actions

```bash
# All actions for an app
npx zapier-sdk list-actions slack --json

# Filter by type: read, write, or search
npx zapier-sdk list-actions slack --action-type write --json

# Get details on a specific action
npx zapier-sdk get-action slack write channel_message --json
```

Action types:
- **write** — create or update data (send a message, create a record)
- **search** — find existing data (look up a user, find a row)
- **read** — list data (list channels, list spreadsheets)

### 4. Inspect Input Fields

Before running an action, check what inputs it needs:

```bash
npx zapier-sdk list-input-fields slack write channel_message \
  --connection-id 12345 --json
```

Some fields are **dynamic** — they only appear after you provide earlier fields. For example, Google Sheets column fields only appear once you specify the spreadsheet:

```bash
npx zapier-sdk list-input-fields google-sheets write add_row \
  --connection-id 12345 \
  --inputs '{"spreadsheet": "abc123", "worksheet": "0"}' \
  --json
```

For dynamic dropdowns (e.g. choosing a channel from a list):

```bash
npx zapier-sdk list-input-field-choices slack write channel_message channel \
  --connection-id 12345 --json
```

For JSON Schema output (useful for agent tool definitions):

```bash
npx zapier-sdk get-input-fields-schema slack write channel_message \
  --connection-id 12345 --json
```

### 5. Run the Action

```bash
npx zapier-sdk run-action slack write channel_message \
  --connection-id 12345 \
  --inputs '{"channel": "C0123ABCD", "text": "Hello from the SDK!"}' \
  --json
```

### 6. Direct API Calls (Going Beyond Pre-built Actions)

When the pre-built actions don't cover your use case, use `curl` to make authenticated HTTP requests directly:

```bash
npx zapier-sdk curl "https://slack.com/api/users.list" \
  --connection-id 12345
```

POST with JSON body:

```bash
npx zapier-sdk curl "https://api.example.com/endpoint" \
  --connection-id 12345 \
  -X POST \
  --json '{"key": "value"}'
```

The connection ID tells Zapier which stored credentials to inject (OAuth tokens, API keys, etc.). No manual header management needed.

**Governance note:** Pre-built actions respect your org's app/action restrictions. Direct API calls via `curl`/`fetch` are not yet governed — direct API governance is on the roadmap.

## Scripting with the CLI

All commands support `--json` for piping. Common pattern — extract a connection ID and use it:

```bash
CONNECTION_ID=$(npx zapier-sdk find-first-connection --app google-sheets --owner me --json 2>/dev/null | jq -r '.data.id')

npx zapier-sdk run-action google-sheets write create_spreadsheet \
  --connection-id $CONNECTION_ID \
  --inputs '{"title": "My Sheet", "headers": ["Name", "Email"]}' \
  --json
```

## TypeScript SDK — For Production Code

When you need something repeatable, embedded, or in production, use the TypeScript SDK. See `references/typescript-sdk.md` for the full API reference.

Quick example:

```typescript
import { createZapierSdk } from "@zapier/zapier-sdk";

const zapier = createZapierSdk();

// Find connection
const { data: conn } = await zapier.findFirstConnection({
  app: "slack", owner: "me", isExpired: false,
});

// Bind connection to app
const slack = zapier.apps.slack({ connectionId: conn.id });

// Run action
const { data: result } = await slack.write.channel_message({
  inputs: { channel: "C0123ABCD", text: "Hello from the SDK!" },
});
```

## Inside Code by Zapier — SDK in a Code Step

The SDK can also run **inside** a JavaScript Code step in any Zap, with full access to your existing app connections — no embedded API keys, no client credentials. This is the right surface when the goal is to collapse a tangle of paths, loops, or sub-zaps into a single Code step inside the same Zap, where the visual editor has become unwieldy.

### Enabling the SDK in a Code step

1. In the Zap editor, add a **Code by Zapier** step → event **Run JavaScript** → **Open in Code Editor**.
2. Click the **Packages** icon in the left sidebar.
3. Toggle **`@zapier/zapier-sdk (latest)`** on. Zapier auto-imports the package and initialises `zapier` at the top of the file.

The SDK is JavaScript only — Python Code steps don't support it. Code triggers (the trigger variant of Code by Zapier) also don't support the SDK; it's actions only.

### Wiring connections

Connections are wired through the UI, not in code:

1. Click **Add app connection** → **+ Add connection**.
2. Search for and select the app.
3. Select an existing connected account from the **Select account connection** dropdown (or connect a new one).
4. The **Account ID Variable** field auto-fills with a readable key based on the account label — for example `slack`, `google_sheets`, `notion`, `chatgpt`. You can edit this to anything (e.g. `slack_personal`, `sheets_reporting`); each key just needs to be unique within the step.

That key becomes the lookup into a runtime `connections` map exposed inside the Code step. You don't see connection IDs anywhere in your code — Zapier resolves the key to the underlying connection at runtime, and credentials never appear in the code, the Zap history, or logs.

Repeat for each app you want the step to call. Only the Zap owner can add or edit connections on the step; collaborators without access to the selected connections can't edit the step.

### Calling actions

Two patterns. Both work; pick whichever reads better for the step.

**Proxy pattern** — bind the connection to an app once, then call actions cleanly:

```javascript
// (zapier is auto-imported — no createZapierSdk() call needed)

const notion = zapier.apps.notion({ connectionId: connections["notion"] });

const { data: page } = await notion.write.create_database_item({
  inputs: {
    database: "DB_ID",
    title: inputData.title,
  },
});

return { pageId: page.id };
```

**Direct `runAction` pattern** — useful when you want a single call, want the action key to be dynamic, or want to keep things flat:

```javascript
const { data } = await zapier.runAction({
  appKey: "notion",
  actionType: "write",
  actionKey: "create_database_item",
  connectionId: connections["notion"],
  inputs: {
    database: "DB_ID",
    title: inputData.title,
  },
});
```

The newer suffix-less parameter names (`app` / `action` / `connection`) also work and are preferred in fresh code outside Code steps; the help article still uses the older `appKey`/`actionKey`/`connectionId` names, which is why the example above mirrors them.

### Direct authenticated HTTP via `zapier.fetch`

When the pre-built actions don't cover the endpoint, call the API directly. Zapier injects the connection's stored credentials (OAuth token, API key, signed headers — whatever the connection uses):

```javascript
const res = await zapier.fetch("https://api.notion.com/v1/databases/DB_ID/query", {
  method: "POST",
  connectionId: connections["notion"],
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ page_size: 50 }),
});
const result = await res.json();
```

`zapier.fetch` is the escape hatch for the long tail of endpoints that don't have a pre-built action.

### Discovering app keys, action keys, and input fields

The Code step UI doesn't tell you what to type. To find the right `appKey`, `actionKey`, and input field names, run the SDK CLI **locally** against the same Zapier account, then transcribe the values into your Code step. The CLI is the canonical discovery tool for this — see the **CLI Workflow** section above. Typical recipe:

```bash
npx zapier-sdk list-connections --owner me --json     # confirm app keys
npx zapier-sdk list-actions notion --json             # find action keys
npx zapier-sdk list-input-fields notion write create_database_item \
  --connection-id 12345 --json                        # find input field names
```

You don't pass the `--connection-id` value into the Code step — it's only needed locally so the CLI can resolve dynamic fields. In the Code step, the value comes from `connections["notion"]`.

### Limits and gotchas

- **JavaScript only.** Python Code steps don't support the SDK.
- **Actions only.** The SDK is not available in Code *triggers*. Triggers / event subscriptions inside Code steps are on the roadmap, not yet shipped.
- **Per-step runtime caps still apply.** Same memory and wall-clock limits as any Code step — up to ~10 minutes on paid plans. Long-running scrapes or sync jobs that exceed this should run externally.
- **Direct `zapier.fetch` calls aren't yet covered by org governance.** Pre-built actions respect your org's app/action restrictions; raw API calls don't yet. Direct API governance is on the roadmap.
- **Credentials never appear in code, Zap history, or logs.** Expired or invalid connections produce a runtime error — re-authenticate in the Zap editor.
- **Owner-only edits.** Only the Zap owner can add or change connections on the step.

### When to prefer this over external SDK use

Reach for SDK-in-Code-step when:

- You're already inside a Zap and the logic has outgrown Paths / Filters / Looping by Zapier — multi-branch routing, nested loops, conditional retries — and the visual editor has become hard to read.
- You'd otherwise build several sub-zaps stitched together with webhooks just to get the control flow right. One Code step with the SDK is usually clearer and cheaper.
- The work is naturally part of *this* Zap's run (triggered by *this* trigger, using *this* run's data) and doesn't need to live in a separate codebase.

Reach for the external SDK / CLI when the work is scheduled, long-running, agentic, lives in a product, or needs a real test/deploy lifecycle.

## Authentication Options

| Method | When to Use | Setup |
|--------|-------------|-------|
| **In a Code step** | Logic that belongs inside a specific Zap | Toggle `@zapier/zapier-sdk (latest)` on in the Code editor's Packages panel; wire connections via the UI. No login or credentials needed. |
| **Browser login** | Local terminal, Claude Code | `npx zapier-sdk login` |
| **Client credentials** | Cowork, CI/CD, servers, headless | `npx zapier-sdk create-client-credentials "my-app"` — save the secret immediately, it's shown only once. Pass via `--credentials-client-id` and `--credentials-client-secret` flags. |
| **Direct token** | Partner OAuth, internal use | Pass token directly to `createZapierSdk()` |

## Zapier Tables

The SDK/CLI can also work with Zapier Tables (structured data storage). See `references/tables.md` for details.

## Common Patterns

**Send yourself a Slack DM:**
```bash
# Look up your Slack user by email
npx zapier-sdk run-action slack search user_by_email \
  --connection-id ID --inputs '{"email": "you@example.com"}' --json

# Send DM using the returned username
npx zapier-sdk run-action slack write direct_message \
  --connection-id ID --inputs '{"channel": "USERNAME", "text": "Hello!"}' --json
```

**Create a Google Sheet and add rows:**
```bash
# Create sheet
npx zapier-sdk run-action google-sheets write create_spreadsheet \
  --connection-id ID --inputs '{"title": "Report", "headers": ["Name", "Score"]}' --json

# Inspect dynamic column fields
npx zapier-sdk list-input-fields google-sheets write add_row \
  --connection-id ID --inputs '{"spreadsheet": "SHEET_ID", "worksheet": "0"}' --json

# Add a row (use the dynamic column keys from above, e.g. COL$A, COL$B)
npx zapier-sdk run-action google-sheets write add_row \
  --connection-id ID \
  --inputs '{"spreadsheet": "SHEET_ID", "worksheet": "0", "COL$A": "Alice", "COL$B": "95"}' \
  --json
```

**Search then act pattern:**
```bash
# Find a contact in HubSpot
npx zapier-sdk run-action hubspot search contact --connection-id ID \
  --inputs '{"email": "alice@example.com"}' --json

# Use the result to update or create something elsewhere
```

## Gotchas and Tips

- **Parameter naming:** As of v0.40.1, parameters use suffix-less names (`--app` not `--app-key`, `--connection` not `--connection-id`). The old names still work but are deprecated.
- **Enterprise/Team plans** are off by default for the SDK — contact Zapier for opt-in.
- **`--json` flag** is your friend — always use it when scripting or piping output.
- **Dynamic fields** are the most common point of confusion. If an action seems to be missing expected inputs, try passing the fields you already know via `--inputs` to `list-input-fields` to reveal the dynamic ones.
- **Connection IDs are numeric** — don't confuse them with app slugs.
- **Timeout default is 180 seconds** for action execution. Override with `--timeout-ms`.
- **Pagination:** Use `--page-size` and `--cursor` for large result sets. The default page size is 100.

## Reference Files

- `references/typescript-sdk.md` — Full TypeScript SDK API reference (methods, types, pagination, app proxy pattern)
- `references/cli-reference.md` — Complete CLI command reference with all flags
- `references/tables.md` — Working with Zapier Tables (create, query, update records)
