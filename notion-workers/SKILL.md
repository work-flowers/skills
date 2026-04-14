---
name: notion-workers
description: >
  Guide for building Notion Workers — small TypeScript programs hosted by Notion
  that extend Custom Agents with custom tool calls and scheduled data syncs. Use
  this skill when the user wants to build a Notion Worker, create custom tools
  that agents can call, set up syncs to pull external data into Notion databases,
  deploy with the `ntn` CLI, configure OAuth or secrets for third-party APIs, or
  troubleshoot Worker execution issues. Also trigger when the user mentions
  `ntn`, `ntn workers`, `@notionhq/workers`, Notion Workers, Worker tools,
  Worker syncs, or wants to give a Notion Custom Agent new capabilities beyond
  what MCP connectors provide. This skill covers the full Worker development
  lifecycle: scaffolding, tool definitions, sync definitions, schema builder,
  authentication (secrets and OAuth), deployment, local testing, monitoring, and
  operational constraints. Prefer this skill over building a standalone Notion
  API integration whenever the goal is to extend a Custom Agent or sync external
  data into Notion on a schedule without self-hosted infrastructure.
---

# Notion Workers

Build and deploy TypeScript programs hosted by Notion that give Custom Agents
new capabilities (Tools) and sync external data into Notion databases (Syncs).

**Status**: Extreme pre-release alpha. Expect breaking changes to the
`@notionhq/workers` package, the `ntn` CLI, and the hosting platform. No SLA,
no uptime guarantee. Support is limited to the GitHub repo at
`makenotion/workers-template`.

**Prerequisite**: The workspace must be on a Notion Business or Enterprise plan,
and a workspace admin must opt in at https://www.notion.so/?target=ai.

---

## Two Capability Types

### Tools
Callable functions that a Custom Agent can invoke on demand during a run. Think
of them as giving the agent a new verb: "send an SMS", "look up the weather",
"query an external API". The agent decides when to call the tool based on its
instructions and the tool's title/description.

### Syncs
Scheduled jobs that pull data from an external source into a Notion database.
One-way only (external → Notion). Think of them as a cron job with a database
attached. Syncs can run from every minute to once a week, or "continuous"
(back-to-back) and "manual" (CLI trigger only).

---

## Build Process

When building a new Worker, follow this sequence. The goal is clarity on scope
before writing code — Workers have hard constraints (timeout, JSON-only returns)
that shape implementation decisions.

### Step 1: Gather requirements

Ask these questions upfront (some may already be answered in conversation):

- **What external service are we connecting to?** Get a link to the API docs.
- **Tool or Sync (or both)?** Tools for agent-callable actions; Syncs for
  scheduled data pulls into a Notion database.
- **For Tools**: What should the agent be able to do? What inputs does it need?
  What should it return?
- **For Syncs**: What data, how often, which Notion database? Does the source
  support cursor-based pagination or incremental fetches?
- **Auth**: Does the external API need an API key (use secrets) or user-level
  OAuth? What scopes are required?
- **Constraints**: Rate limits, payload sizes, expected execution time (must
  complete within 30–60 seconds).

### Step 2: Review the external API documentation

Before scaffolding, fetch and read the API docs. Extract:

- **Auth type and credential fields**
- **Relevant endpoints** — URL patterns, methods, params, response shapes
- **Pagination** — cursor-based, offset/limit, page tokens
- **Rate limits** — requests per minute/hour, retry-after headers
- **Response sizes** — will responses fit within the timeout window?

### Step 3: Propose an implementation plan

Summarise and confirm with the user:

- Tool name(s) and/or sync key(s)
- Auth approach (secrets vs. OAuth) and which credentials are needed
- For syncs: schedule, sync mode (replace vs. incremental), batch strategy
- Any non-obvious design decisions
- Whether secrets need to be stored in 1Password (use the 1password-cli skill
  for credential retrieval patterns)

Get sign-off, then scaffold.

---

## Quick Reference

| Concept | What It Does | Key Command |
|---------|-------------|-------------|
| Scaffold | Create a new Worker project | `ntn workers new` |
| Deploy | Push to Notion's hosting | `ntn workers deploy` |
| Test locally | Run a tool without deploying | `ntn workers exec <toolName> --local -d '{"key":"value"}'` |
| Secrets | Store API keys | `ntn workers env set KEY=value` |
| Pull secrets | Create local `.env` from stored secrets | `ntn workers env pull` |
| OAuth setup | Configure third-party OAuth | `worker.oauth(...)` in code |
| OAuth redirect | Get the redirect URL for provider config | `ntn workers oauth show-redirect-url` |
| OAuth start | Begin the OAuth flow | `ntn workers oauth start <name>` |
| Sync status | Live-updating sync health | `ntn workers sync status` |
| Sync trigger | Run a sync immediately | `ntn workers sync trigger <key>` |
| Sync preview | Preview output without writing to DB | `ntn workers sync trigger <key> --preview` |
| Sync reset | Restart from scratch | `ntn workers sync state reset <key>` |
| Capabilities | List all tools and syncs | `ntn workers capabilities list` |
| Pause/resume | Disable or enable a capability | `ntn workers capabilities disable/enable <key>` |
| Logs | View execution logs | `ntn workers runs logs <runId>` |
| Login | Authenticate the CLI | `ntn login` |

---

## Project Setup

```bash
# Install the ntn CLI globally
npm i -g ntn

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
│   └── index.ts          # Worker entry point — define tools and syncs here
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

worker.tool("toolName", {
  title: "Human-Readable Title",
  description: "Tell the agent when and why to use this tool.",
  schema: j.object({
    param1: j.string().describe("What this parameter is for."),
    param2: j.number().describe("Numeric input.").nullable(),
  }),
  execute: async ({ param1, param2 }) => {
    // Call external API, process data, return result
    const response = await fetch("https://api.example.com/endpoint", {
      headers: { Authorization: `Bearer ${process.env.API_KEY}` },
    });
    if (!response.ok) throw new Error(`API error: ${response.statusText}`);
    const data = await response.json();
    return JSON.stringify(data);
  },
});
```

**Key rules for tools:**

- The `description` field is what the agent reads to decide whether to call the
  tool. Write it as if explaining to a capable colleague what this tool does
  and when it's useful.
- Use the schema builder `j` for inputs. It auto-sets `required` and
  `additionalProperties`. Use `.nullable()` for optional fields.
- The `execute` function must return a string or JSON-serialisable value.
- Tools must complete within 30–60 seconds (platform timeout).
- Never throw raw errors to the caller. Return structured error responses:
  `{ success: false, error: "message" }`.
- One tool = one concern. Keep tools focused.

### Schema Builder Reference

```typescript
import { j } from "@notionhq/workers/schema-builder";

// String
j.string().describe("Description")

// Number
j.number().describe("Description")

// Boolean
j.boolean().describe("Description")

// Optional (nullable)
j.string().describe("Description").nullable()

// Object
j.object({
  field1: j.string().describe("..."),
  field2: j.number().describe("..."),
})

// Enum
j.enum(["option1", "option2"]).describe("Choose one")

// Array
j.array(j.string()).describe("List of strings")
```

---

## Defining Syncs

For detailed sync patterns (incremental, replace, cursor management, batch
pagination), read `references/syncs.md` in this skill folder.

A sync pulls external data into a Notion database on a schedule:

```typescript
worker.sync("syncKey", {
  title: "Human-Readable Sync Name",
  description: "What this sync does and where data comes from.",
  schedule: "1h",  // How often: "1m" to "7d", "continuous", or "manual"
  mode: "replace",  // "replace" or "incremental"
  schema: {
    properties: [
      { name: "Name", type: "title" },
      { name: "Email", type: "email" },
      { name: "Amount", type: "number" },
      { name: "Status", type: "select", options: ["Active", "Inactive"] },
      { name: "Last Updated", type: "date" },
    ],
  },
  execute: async ({ state }) => {
    // Fetch data from external source
    const response = await fetch("https://api.example.com/data");
    const items = await response.json();

    return {
      changes: items.map(item => ({
        id: item.id,  // Unique external ID for upsert matching
        properties: {
          "Name": item.name,
          "Email": item.email,
          "Amount": item.amount,
          "Status": item.status,
          "Last Updated": item.updatedAt,
        },
      })),
      hasMore: false,
      nextState: state,
    };
  },
});
```

**Key sync concepts:**

- **Schedule options**: `"1m"`, `"5m"`, `"15m"`, `"30m"`, `"1h"`, `"6h"`,
  `"12h"`, `"1d"`, `"7d"`, `"continuous"`, `"manual"`
- **Modes**: `"replace"` overwrites the full dataset each run. `"incremental"`
  uses cursor state to fetch only new/changed records.
- **State management**: The `state` object persists between runs. Use it to
  store cursors, timestamps, or page tokens for incremental syncs.
- **Batching**: Return `hasMore: true` with a `nextState` to paginate. The
  platform will call `execute` again with the new state. Keep batches around
  100 records to avoid timeouts.
- **Deploying does NOT reset sync state.** Use `ntn workers sync state reset
  <key>` to restart from scratch.
- **One-way only**: External → Notion. No reverse sync.

---

## Authentication

### Secrets (API keys, tokens)

For services that use static API keys or tokens:

```bash
# Store a secret
ntn workers env set API_KEY=your-secret-here

# Store multiple secrets
ntn workers env set TWILIO_SID=ACxxxxxxx
ntn workers env set TWILIO_TOKEN=your-auth-token

# Pull secrets to a local .env file for development
ntn workers env pull
```

Access in code via `process.env`:

```typescript
const apiKey = process.env.API_KEY;
```

**Integration with 1Password**: For credential management across projects, use
the 1password-cli skill to retrieve secrets from 1Password and set them in the
Worker:

```bash
ntn workers env set API_KEY=$(op read "op://Work/ServiceName/api-key")
```

### OAuth

For services requiring user authorisation (Google, GitHub, etc.), read
`references/oauth.md` in this skill folder for the full setup flow.

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

After deploying:

```bash
# Get the redirect URL — add this to your OAuth provider's app settings
ntn workers oauth show-redirect-url

# Start the OAuth flow
ntn workers oauth start githubAuth
```

Use the token in tools:

```typescript
const token = await githubAuth.accessToken();
```

---

## Deployment & Testing

```bash
# Type-check
npm run check

# Build
npm run build

# Deploy to Notion
ntn workers deploy

# Test a tool locally (without deploying)
ntn workers exec sayHello --local -d '{"name": "Dennis"}'

# Test a tool on the deployed Worker
ntn workers exec sayHello -d '{"name": "Dennis"}'
```

After deploying, add the tool to a Custom Agent in Notion:
1. Open the agent editor
2. Add a custom tool call
3. Select the deployed Worker and the specific tool

---

## Operational Constraints

These are hard boundaries that shape implementation decisions:

| Constraint | Detail |
|------------|--------|
| **Timeout** | 30–60 seconds per execution. For large datasets, paginate using `hasMore` / `nextState`. |
| **Return format** | JSON only. No `Date` objects, `Map`, `Set`, etc. |
| **Error handling** | Never throw to the caller. Return `{ success: false, error: "..." }`. |
| **Sync direction** | One-way: external → Notion. No reverse sync. |
| **Runtime** | Node.js/TypeScript. No Bun-specific APIs (`Bun.file()`, `Bun.serve()`, etc.). |
| **Observability** | Logs via `ntn workers runs logs <runId>`. No dashboard, no alerting, no retry visibility during alpha. |
| **Idempotency** | Design tools and syncs to be safely re-runnable. External side effects (sending SMS, posting messages) should be guarded if appropriate. |

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
| Agent needs to call an external API | ✅ Best fit — tool appears in agent's toolkit | Possible via webhook, but indirect | Overkill for agent integration |
| Scheduled one-way data sync | ✅ Good fit if source is niche/custom | ✅ Better for commodity integrations with existing connectors | ✅ If you need full control |
| Multi-step orchestration across services | ❌ Single-purpose, 60s timeout | ✅ Built for this | Depends on complexity |
| Bi-directional sync | ❌ One-way only | ✅ Can build both directions | ✅ Full control |
| High reliability / SLA needed | ❌ Alpha, no guarantees | ✅ Mature platform | ✅ You control uptime |
| No infrastructure to maintain | ✅ Notion hosts it | ✅ Zapier hosts it | ❌ You host it |

---

## Reference Files

For deeper topics, read these reference files in this skill folder:

- `references/syncs.md` — Detailed sync patterns: incremental vs. replace,
  cursor management, backfill strategies, batch pagination, schema design for
  Notion database properties, and troubleshooting common sync issues.
- `references/oauth.md` — Full OAuth setup flow: provider configuration,
  redirect URLs, token refresh handling, and patterns for Google, GitHub, and
  generic OAuth2 providers.

---

## Further Reading

- Workers template repo: https://github.com/makenotion/workers-template
- Notion Custom Agents docs: https://www.notion.com/help/custom-agents
- Notion Dev Slack: https://join.slack.com/t/notiondevs/
