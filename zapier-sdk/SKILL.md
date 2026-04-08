---
name: zapier-sdk
description: "Guide for using the Zapier SDK and CLI to explore apps, discover actions, and run authenticated API calls against 9,000+ connected apps. Use this skill whenever the user mentions the Zapier SDK, zapier-sdk, @zapier/zapier-sdk, wants to call an external app's API through Zapier's auth infrastructure, needs to explore what actions an app supports, wants to run a one-off action via CLI (e.g. send a Slack message, create a Google Sheet, look up a HubSpot contact), or asks about making authenticated fetch/curl requests through Zapier. Also trigger when the user asks about Zapier connections, listing connected apps, or running actions against their Zapier-connected accounts. This skill is about CONSUMING Zapier's pre-built connectors (calling Slack, Google Sheets, Salesforce, etc. through Zapier's auth layer) -- NOT about BUILDING new Zapier integrations (that's the zapier-platform-cli and zapier-platform-ui skills instead)."
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
- Logged in: `npx zapier-sdk login` (opens browser)
- At least one connected app at https://zapier.com/app/assets/connections

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

## Authentication Options

| Method | When to Use | Setup |
|--------|-------------|-------|
| **Browser login** | Local dev, CLI exploration | `npx zapier-sdk login` |
| **Client credentials** | CI/CD, servers, serverless | `npx zapier-sdk create-client-credentials "my-app"` — save the secret immediately, it's shown only once |
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
