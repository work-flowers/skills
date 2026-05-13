---
name: notion-workers
description: >
  Guide for building Notion Workers — TypeScript programs hosted by Notion that
  extend Custom Agents with custom tools, sync external data into Notion
  databases, and receive webhook events from third-party services. Use when the
  user wants to build a Worker, create tools agents can call, set up syncs to
  pull external data into Notion, receive webhooks from external services,
  deploy with the ntn CLI, or configure OAuth/secrets for third-party APIs.
  Also trigger when the user mentions ntn, ntn workers, notionhq/workers,
  Notion Workers, Worker tools, Worker syncs, Worker webhooks, or wants to
  give a Custom Agent new capabilities beyond MCP connectors. Covers the full
  lifecycle: scaffolding, capability definitions, schema and builders,
  authentication, deployment, local testing, monitoring, and constraints.
  Prefer this over a standalone Notion API integration when the goal is
  extending a Custom Agent, receiving webhooks, or syncing external data
  without self-hosted infrastructure.
---

# Notion Workers

Build and deploy TypeScript programs hosted by Notion that give Custom Agents
new capabilities (Tools), sync external data into Notion databases (Syncs),
receive HTTP events from external services (Webhooks), and run custom actions
from database buttons (Automations).

**Status**: Pre-release. Expect breaking changes to the `@notionhq/workers`
package, the `ntn` CLI, and the hosting platform. No SLA, no uptime guarantee.
Official docs at https://developers.notion.com/workers and community support
in the Notion Devs Slack.

**Prerequisite**: The workspace must be on a Notion Business or Enterprise plan,
and a workspace admin must opt in at https://www.notion.so/?target=ai.

---

## Capability Types

A Worker exports a single `Worker` instance and registers one or more
capabilities on it.

### Tools
Callable functions that a Custom Agent invokes on demand. Think of them as
giving the agent a new verb: "look up a customer", "create a Jira ticket",
"send an SMS". The agent decides when to call based on the tool's description.

### Syncs
Scheduled jobs that pull external data into a Notion database managed by the
Worker. One-way only (external → Notion). Modes: `replace` (full snapshot,
mark-and-sweep deletes) and `incremental` (deltas only, explicit deletes).

### Webhooks
HTTP endpoints exposed by the Worker. Use them to receive events pushed from
external services (GitHub, Stripe, Zendesk, etc.). Notion creates a unique URL
per webhook capability after deploy.

### Automations
Custom actions triggered from a Notion database button or automation rule.
Receives the triggering page plus a pre-authenticated Notion API client. Notion's
documentation is still under construction — see
https://developers.notion.com/workers/guides/automations for the latest.

---

## Build Process

When building a new Worker, follow this sequence. The goal is clarity on scope
before writing code — Workers have hard constraints (timeout, JSON-only returns)
that shape implementation decisions.

### Step 1: Gather requirements

Ask these questions upfront (some may already be answered in conversation):

- **What external service are we connecting to?** Get a link to the API docs.
- **Which capability types?** Tool (agent-callable), Sync (scheduled pull),
  Webhook (incoming push), Automation (Notion button trigger), or a combination.
- **For Tools**: What should the agent be able to do? Inputs? Output shape?
  Read-only or does it mutate state?
- **For Syncs**: What data, how often, into what Notion database? Does the
  source support cursors / change feeds for incremental?
- **For Webhooks**: What provider? Does it sign requests? What's the event shape?
- **Auth**: API key (secret) or user-level OAuth? What scopes?
- **Constraints**: Rate limits, payload sizes, expected execution time.

### Step 2: Review the external API documentation

Before scaffolding, fetch and read the relevant API docs. Extract:

- **Auth type and credential fields**
- **Relevant endpoints** — URL patterns, methods, params, response shapes
- **Pagination** — cursor-based, offset/limit, page tokens
- **Rate limits** — requests per minute/hour, retry-after headers
- **Webhook signing** (if applicable) — header names, hashing algorithm

### Step 3: Propose an implementation plan

Summarise and confirm with the user:

- Capability keys (tool name(s), sync key(s), webhook key(s))
- For syncs: which `worker.database()` declarations are needed and the schema
- Auth approach (secrets vs. OAuth) and which credentials are needed
- For syncs: schedule, mode (`replace` vs. `incremental`), pagination strategy
- Whether a `worker.pacer()` is needed for rate-limiting outbound calls
- Whether secrets need to be stored in 1Password (use the 1password-cli skill
  for credential retrieval patterns)

Get sign-off, then scaffold.

---

## Quick Reference

| Concept | What It Does | Key Command |
|---------|--------------|-------------|
| Install CLI | Install `ntn` on macOS/Linux | `curl -fsSL https://ntn.dev \| bash` |
| Login | Authenticate the CLI | `ntn login` |
| Scaffold | Create a new Worker project | `ntn workers new` |
| Deploy | Push to Notion's hosting | `ntn workers deploy` |
| Test tool locally | Run a tool without deploying | `ntn workers exec <toolName> --local -d '{"key":"value"}'` |
| Run hosted tool | Run a deployed tool | `ntn workers exec <toolName> -d '{"key":"value"}'` |
| Set secrets | Store one or more env vars | `ntn workers env set KEY=value KEY2=value2` |
| List secrets | List keys (no values) | `ntn workers env list` |
| Pull secrets | Write remote secrets to local `.env` | `ntn workers env pull` |
| Push secrets | Push local `.env` to remote | `ntn workers env push` |
| OAuth redirect URL | Show URL for provider config | `ntn workers oauth show-redirect-url` |
| Start OAuth flow | Begin browser-based authorisation | `ntn workers oauth start <capabilityKey>` |
| Sync status | Live-updating dashboard | `ntn workers sync status` |
| Sync preview | Run without writing to the DB | `ntn workers sync trigger <key> --preview` |
| Sync trigger | Run immediately | `ntn workers sync trigger <key>` |
| Sync state reset | Restart from scratch | `ntn workers sync state reset <key>` |
| Sync state inspect | View current cursor/state | `ntn workers sync state get <key>` |
| Webhook URLs | List webhook URLs for a deploy | `ntn workers webhooks list` |
| Capabilities | List tools, syncs, webhooks | `ntn workers capabilities list` |
| Pause / resume | Disable / enable a capability | `ntn workers capabilities disable/enable <key>` |
| Runs | List recent executions | `ntn workers runs list` |
| Logs | View execution logs | `ntn workers runs logs <runId>` |

---

## Project Setup

```bash
# Install the ntn CLI (recommended)
curl -fsSL https://ntn.dev | bash

# Or via npm (requires Node 22+ / npm 10+)
npm install --global ntn

# Authenticate
ntn login

# Scaffold a new Worker
ntn workers new
# Follow the prompts, then cd into the project directory

# Install dependencies
npm install
```

The scaffolded project contains:

```
my-worker/
├── src/
│   └── index.ts          # Worker entry point — define capabilities here
├── workers.json          # Worker ID for this project
├── package.json
└── tsconfig.json
```

---

## Defining Tools

A tool gives a Custom Agent a new capability. Define tools in `src/index.ts`:

```typescript
import { Worker } from "@notionhq/workers";
import { j } from "@notionhq/workers/schema-builder";

const worker = new Worker();
export default worker;

worker.tool("lookupCustomer", {
  title: "Lookup Customer",
  description: "Find a customer by email address.",
  schema: j.object({
    email: j.email().describe("The customer's email address."),
  }),
  hints: { readOnlyHint: true },
  execute: async ({ email }, { notion }) => {
    const customer = await findCustomerByEmail(email);
    if (!customer) {
      return { found: false, message: `No customer found for ${email}.` };
    }
    return {
      found: true,
      name: customer.name,
      plan: customer.plan,
    };
  },
});
```

**Key rules for tools:**

- The `description` is what the agent reads to decide whether to call the tool.
  Write it as an instruction boundary: what it does and *when* to use it.
  Narrow descriptions ("Create a support ticket when the user asks to escalate
  an issue") help the agent choose correctly.
- Use the `j` schema builder for inputs. It auto-sets `required` and
  `additionalProperties: false`. Use `.nullable()` for optional fields.
- The second `execute` argument is a context object. `context.notion` is a
  pre-authenticated Notion SDK client (see "Using Notion from a Worker" below).
- `hints: { readOnlyHint: true }` marks the tool as safe to auto-execute.
  Write tools without this hint will prompt the user for permission.
- Optional `outputSchema: j.object({...})` validates the return value.
- Return JSON-serialisable values. Never throw raw errors to the agent;
  return structured `{ success: false, error: "..." }` shapes instead.

### Tool input schema (the `j` builder)

`j` from `@notionhq/workers/schema-builder` builds JSON Schema for tool I/O.
Use `.describe()` on every field — descriptions tell the agent what each value
means.

```typescript
import { j } from "@notionhq/workers/schema-builder";

j.string().describe("Search query.")
j.number().describe("Maximum results.")
j.integer().describe("Whole number.")
j.boolean().describe("Whether to include archived items.")
j.email().describe("Email address.")
j.uuid().describe("External record ID.")
j.date().describe("YYYY-MM-DD")
j.datetime().describe("ISO 8601 timestamp")
j.enum("low", "medium", "high").describe("Priority.")
j.array(j.string())
j.array(j.string(), { minItems: 1 })
j.anyOf(j.string(), j.number())
j.string().nullable()           // optional field; still "required" in JSON Schema, value may be null
j.object({ a: j.string() })     // sets required + additionalProperties: false
```

For the full list (`j.hostname()`, `j.ipv4()`, `j.duration()`, `j.ref()`, etc.)
see https://developers.notion.com/workers/reference/schema.

---

## Defining Syncs

Syncs are a two-step pattern: declare a managed database with `worker.database()`,
then attach one or more syncs to it with `worker.sync()`. For detailed sync
patterns (incremental, replace, cursors, backfill+delta, batch pagination),
read `references/syncs.md` in this skill folder.

```typescript
import { Worker } from "@notionhq/workers";
import * as Schema from "@notionhq/workers/schema";
import * as Builder from "@notionhq/workers/builder";

const worker = new Worker();
export default worker;

const tasks = worker.database("tasks", {
  type: "managed",
  initialTitle: "Tasks",
  primaryKeyProperty: "Task ID",
  schema: {
    properties: {
      Name: Schema.title(),
      "Task ID": Schema.richText(),
      Status: Schema.select([
        { name: "Open" },
        { name: "Done", color: "green" },
      ]),
      "Updated At": Schema.date(),
    },
  },
});

worker.sync("tasksSync", {
  database: tasks,
  mode: "replace",
  schedule: "30m",
  execute: async (state) => {
    const page = state?.page ?? 1;
    const { items, hasMore } = await fetchTasks(page);
    return {
      changes: items.map((item) => ({
        type: "upsert" as const,
        key: item.id,
        properties: {
          Name: Builder.title(item.name),
          "Task ID": Builder.richText(item.id),
          Status: Builder.select(item.status),
          "Updated At": Builder.date(item.updatedAt.slice(0, 10)),
        },
      })),
      hasMore,
      nextState: hasMore ? { page: page + 1 } : undefined,
    };
  },
});
```

**Key sync concepts:**

- **Three schema imports**:
  - `j` from `@notionhq/workers/schema-builder` — tool I/O only
  - `Schema` from `@notionhq/workers/schema` — database property definitions
  - `Builder` from `@notionhq/workers/builder` — property values in sync changes
- **`worker.database()`** declares a managed database. Notion creates and migrates
  it on each deploy. `primaryKeyProperty` is the column that holds the upstream
  ID — must exist in `schema.properties`.
- **Sync changes** use `{ type: "upsert" | "delete", key, properties }`. The `key`
  matches the value of the primary key property.
- **Replace mode** does mark-and-sweep: rows not seen by the end of a cycle
  (`hasMore: false`) are deleted automatically.
- **Incremental mode** only touches rows mentioned in `changes`. Deletes must be
  explicit (`{ type: "delete", key }`).
- **Schedule**: `"5m"`, `"15m"`, `"30m"`, `"1h"`, `"6h"`, `"12h"`, `"1d"`,
  `"7d"`, `"continuous"`, or `"manual"`. Minimum `5m`, maximum `7d`. Default
  is every 30 minutes.
- **State persists across deploys.** Use `ntn workers sync state reset <key>`
  to restart from scratch (e.g., after a schema change).
- **Pacer**: declare with `worker.pacer()` and call `await pacer.wait()` before
  outbound requests to spread the rate-limit budget over time. Multiple
  capabilities sharing a pacer get the budget split automatically.

---

## Defining Webhooks

A webhook capability exposes an HTTP URL that an external service can call.
For full coverage (signature verification, retries, idempotency), read
`references/webhooks.md`.

```typescript
import { Worker, WebhookVerificationError } from "@notionhq/workers";

const worker = new Worker();
export default worker;

worker.webhook("onGithubPush", {
  title: "GitHub Push Webhook",
  description: "Handles push events from GitHub repositories.",
  execute: async (events) => {
    for (const event of events) {
      // event has: deliveryId, body (parsed JSON), rawBody, headers, method
      console.log("Delivery:", event.deliveryId);
      console.log("Body:", event.body);
    }
  },
});
```

After deploy, get the URLs:

```bash
ntn workers deploy
ntn workers webhooks list
```

Treat the URL as a secret — anyone with it can post events. For providers that
sign requests, verify in code using `event.rawBody` and `event.headers`, and
throw `WebhookVerificationError` on failure. Five consecutive verification
failures cause Notion to block the webhook (redeploy to reset).

If the handler throws a non-verification error, Notion retries up to 3 times.

---

## Authentication

### Secrets (API keys, tokens)

For services that use static API keys or tokens:

```bash
# Store one or more secrets
ntn workers env set API_KEY=your-secret
ntn workers env set TWILIO_SID=ACxxx TWILIO_TOKEN=auth-token

# Pull secrets to a local .env (for local testing)
ntn workers env pull

# Push local .env back up (after editing locally)
ntn workers env push

# List keys without values
ntn workers env list

# Remove a secret
ntn workers env unset API_KEY
```

Access in code via `process.env`:

```typescript
const apiKey = process.env.API_KEY;
if (!apiKey) throw new Error("API_KEY is not configured");
```

**Integration with 1Password**: For credential management across projects, use
the 1password-cli skill to retrieve secrets from 1Password and set them in the
Worker:

```bash
ntn workers env set API_KEY=$(op read "op://Work/ServiceName/api-key")
```

### OAuth

For services requiring user authorisation (Google, GitHub, Salesforce, etc.),
read `references/oauth.md` for the full setup flow.

Quick overview:

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

Setup is deploy → get redirect URL → configure provider → store secrets →
redeploy → authorise:

```bash
ntn workers deploy
ntn workers oauth show-redirect-url
# Add the URL to your OAuth provider's app settings, copy client ID/secret
ntn workers env set GITHUB_CLIENT_ID=xxx GITHUB_CLIENT_SECRET=yyy
ntn workers deploy
ntn workers oauth start githubAuth     # NB: pass the capability key, not the `name` field
```

Use the token in any capability:

```typescript
const token = await githubAuth.accessToken();
```

The runtime refreshes tokens automatically.

---

## Using Notion from a Worker

Every capability's `execute` receives `context.notion`, a pre-authenticated
Notion API client (the official `@notionhq/client` SDK):

```typescript
worker.tool("getPageTitle", {
  title: "Get Page Title",
  description: "Read the title of a Notion page.",
  schema: j.object({ pageId: j.string().describe("Notion page ID.") }),
  hints: { readOnlyHint: true },
  execute: async ({ pageId }, { notion }) => {
    const page = await notion.pages.retrieve({ page_id: pageId });
    return page;
  },
});
```

How `context.notion` is authenticated depends on the capability:

| Capability | How it works |
|------------|--------------|
| Tool called by a Custom Agent (hosted) | Token set automatically. Same permissions as the agent. No setup. |
| Sync, webhook, automation | You must provide `NOTION_API_TOKEN` as a secret. |
| Local testing (`--local`) | You must provide `NOTION_API_TOKEN` in your local `.env`. |

For non-tool capabilities or local testing, create either a [personal access
token](https://developers.notion.com/guides/get-started/personal-access-tokens)
(acts as you) or an internal integration (acts as a bot, must be explicitly
connected to each page) and store it:

```bash
ntn workers env set NOTION_API_TOKEN=ntn_...
```

For internal integrations, open each page/database the Worker needs and add the
integration under Connections.

---

## Deployment & Testing

```bash
# Type-check
npm run check

# Build
npm run build

# Deploy to Notion
ntn workers deploy

# Test a tool locally (without deploying). Loads .env by default.
ntn workers exec lookupCustomer --local -d '{"email": "ada@example.com"}'

# Use a different env file
ntn workers exec lookupCustomer --local --dotenv .env.local -d '{...}'

# Skip env loading
ntn workers exec lookupCustomer --local --no-dotenv -d '{...}'

# Test the deployed tool
ntn workers exec lookupCustomer -d '{"email": "ada@example.com"}'

# Preview a sync without writing to the DB
ntn workers sync trigger tasksSync --preview
```

After deploying, attach the Worker's tools to a Custom Agent in Notion:
1. Open the agent editor
2. Add a custom tool call
3. Select the deployed Worker and the specific tool

---

## Operational Constraints

These are hard boundaries that shape implementation decisions:

| Constraint | Detail |
|------------|--------|
| **Timeout** | Per-execution timeout enforced by the platform. For large datasets, paginate with `hasMore` / `nextState`. Keep batches around 100 records. |
| **Return format** | JSON-serialisable only. No `Date` objects, `Map`, `Set`, etc. |
| **Tool error handling** | Don't throw raw errors to the agent. Return structured `{ success: false, error: "..." }`. |
| **Webhook error handling** | Throw `WebhookVerificationError` on signature failure (no retry). Other thrown errors → up to 3 retries. |
| **Sync direction** | One-way only: external → Notion. No reverse sync. |
| **Managed DB schema** | Code defines the schema; properties defined in code aren't user-editable in Notion. Users can still add their own properties. Schema changes can drop data on deploy. |
| **Schedule range** | 5 minutes minimum to 7 days maximum, plus `"continuous"` and `"manual"`. |
| **Webhook URL** | Treat as a secret — verify provider signatures inside the handler. |
| **Runtime** | Node.js/TypeScript sandbox. No Bun-specific APIs. |
| **Observability** | `ntn workers runs list/logs <runId>` and `ntn workers sync status`. No dashboard during pre-release. |

### Sync health statuses

| Status | Meaning |
|--------|---------|
| HEALTHY | Last run succeeded |
| INITIALIZING | Deployed but hasn't completed a run yet |
| WARNING | 1–2 consecutive failures |
| ERROR | 3+ consecutive failures |
| DISABLED | Paused via `ntn workers capabilities disable` |

---

## When to Use Workers vs. Alternatives

| Scenario | Use Workers | Use Zapier | Use Notion API (self-hosted) |
|----------|-------------|-----------|------------------------------|
| Agent needs to call an external API | ✅ Best fit — tool appears in the agent's toolkit | Possible via webhook, but indirect | Overkill |
| Scheduled one-way data sync | ✅ Good fit if source is niche/custom | ✅ Better for commodity integrations | ✅ If you need full control |
| Receive webhooks from a service | ✅ First-class capability | ✅ Built-in catch hooks | ✅ Build your own endpoint |
| Multi-step orchestration across services | ❌ Single-purpose per capability | ✅ Built for this | Depends on complexity |
| Bi-directional sync | ❌ One-way only | ✅ Can build both directions | ✅ Full control |
| High reliability / SLA needed | ❌ Pre-release, no guarantees | ✅ Mature platform | ✅ You control uptime |
| No infrastructure to maintain | ✅ Notion hosts it | ✅ Zapier hosts it | ❌ You host it |

---

## Reference Files

For deeper topics, read these reference files in this skill folder:

- `references/syncs.md` — Detailed sync patterns: `worker.database()` setup,
  `Schema` and `Builder` helpers, replace vs. incremental mode, cursor
  management, backfill+delta pattern, cross-database relations, pacers,
  troubleshooting, and migration notes for anyone moving from the older
  inline-schema sync API.
- `references/webhooks.md` — Webhook deep dive: event object shape, signature
  verification (GitHub, Stripe, generic HMAC), retries, idempotency, and using
  the Notion API from a webhook handler.
- `references/oauth.md` — Full OAuth setup flow: provider configuration,
  redirect URLs, token refresh handling, and patterns for Google, GitHub,
  Salesforce, and generic OAuth2 providers.

---

## Further Reading

- Official docs: https://developers.notion.com/workers
- SDK reference: https://developers.notion.com/workers/reference/sdk
- Schema & builders: https://developers.notion.com/workers/reference/schema
- CLI command reference: https://developers.notion.com/cli/reference/commands
- Notion Custom Agents: https://www.notion.com/help/custom-agents
- Notion Devs Slack: https://join.slack.com/t/notiondevs/
