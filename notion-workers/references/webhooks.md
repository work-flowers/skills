# Webhooks Reference

Detailed guidance on building Worker webhooks — HTTP endpoints exposed by your
Worker that external services can post events to.

---

## Table of Contents

1. [When to Use Webhooks](#when-to-use-webhooks)
2. [The Event Object](#the-event-object)
3. [Webhook URLs](#webhook-urls)
4. [Signature Verification](#signature-verification)
5. [Provider Examples](#provider-examples)
6. [Retries and Failure Handling](#retries-and-failure-handling)
7. [Idempotency](#idempotency)
8. [Using Notion from a Webhook](#using-notion-from-a-webhook)
9. [Troubleshooting](#troubleshooting)

---

## When to Use Webhooks

| Approach | When to Use |
|----------|-------------|
| **Webhook** | External service pushes events on its own schedule (GitHub push, Stripe payment, Zendesk ticket update, Calendly booking, etc.) |
| **Sync** | You pull data from the external service on a Notion-controlled schedule. Use this when the source has no webhook, or when you want a consistent point-in-time snapshot. |
| **Tool** | A Custom Agent calls the external service on demand. Use when the trigger is user intent in Notion, not an external event. |

Webhooks and syncs can coexist. A common pattern: a webhook fires Notion
updates immediately on event, and a low-frequency sync sweeps in case any
webhooks were missed.

---

## The Event Object

The `execute` function receives an array of `WebhookEvent` objects. Currently
the array holds one event per call, but future versions may batch.

```typescript
worker.webhook("onExternalEvent", {
  title: "External Event Handler",
  description: "Processes incoming webhook requests.",
  execute: async (events) => {
    for (const event of events) {
      // event.deliveryId, event.body, event.rawBody, event.headers, event.method
    }
  },
});
```

| Property | Type | Description |
|----------|------|-------------|
| `deliveryId` | `string` | Unique ID for this Notion delivery. Stable across Notion-side retries of the same inbound HTTP request. |
| `body` | `Record<string, unknown>` | Parsed JSON body. `{}` if the body wasn't valid JSON. |
| `rawBody` | `string` | Original request body as a string. Use for signature verification. |
| `headers` | `Record<string, string>` | Request headers. Names are lowercased. |
| `method` | `string` | HTTP method. Webhook URLs accept `POST`. |

Note: `deliveryId` is stable across Notion's worker retries, but if the
**provider** resends the same event (e.g. because they didn't get a 200 back
in time), Notion treats it as a new delivery with a new `deliveryId`. For true
idempotency, use the provider's event ID from the payload.

---

## Webhook URLs

After deploy, Notion creates one URL per webhook capability:

```
https://www.notion.so/webhooks/worker/{spaceId}/{workerId}/{uniqueWebhookId}/{webhookName}
```

The `uniqueWebhookId` segment acts as a shared secret. Treat the URL as
sensitive — anyone with it can post events. Add provider signature
verification inside the handler for any externally exposed webhook.

Get the URLs:

```bash
ntn workers webhooks list
ntn workers webhooks list --json     # for scripts
ntn workers webhooks list --plain    # tab-separated, grep-friendly
```

---

## Signature Verification

When the provider signs requests with a shared secret, verify each request and
throw `WebhookVerificationError` on failure. Notion records verification
failures separately from other errors:

- A `WebhookVerificationError` is **not** retried.
- After 5 consecutive verification failures, Notion blocks the webhook before
  even running the handler. Redeploy to reset.
- Successful runs reset the consecutive failure counter.

```typescript
import { WebhookVerificationError } from "@notionhq/workers";
```

Always use `event.rawBody` (not `event.body`) for HMAC computation — JSON
re-stringification can change whitespace and break the signature.

Use `crypto.timingSafeEqual` for the final comparison to avoid timing attacks.

---

## Provider Examples

### GitHub (HMAC-SHA256, `X-Hub-Signature-256`)

```typescript
import * as crypto from "node:crypto";
import { Worker, WebhookVerificationError } from "@notionhq/workers";

const worker = new Worker();
export default worker;

function verifyGitHub(rawBody: string, headers: Record<string, string>): void {
  const secret = process.env.GITHUB_WEBHOOK_SECRET;
  if (!secret) {
    throw new WebhookVerificationError("GITHUB_WEBHOOK_SECRET not configured");
  }

  const signature = headers["x-hub-signature-256"];
  if (!signature?.startsWith("sha256=")) {
    throw new WebhookVerificationError("Invalid GitHub signature");
  }

  const expected = `sha256=${crypto
    .createHmac("sha256", secret)
    .update(rawBody)
    .digest("hex")}`;

  if (signature.length !== expected.length) {
    throw new WebhookVerificationError("Invalid GitHub signature");
  }

  if (!crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected))) {
    throw new WebhookVerificationError("Invalid GitHub signature");
  }
}

worker.webhook("onGithubPush", {
  title: "GitHub Push",
  description: "Handles push events from GitHub repositories.",
  execute: async (events) => {
    for (const event of events) {
      verifyGitHub(event.rawBody, event.headers);
      console.log("Verified GitHub event:", event.body);
    }
  },
});
```

Set the secret before deploying (you can copy it from the GitHub repo's
webhook settings page):

```bash
ntn workers env set GITHUB_WEBHOOK_SECRET=your-secret
```

### Stripe (HMAC-SHA256, `Stripe-Signature` with timestamp)

Stripe's signature is `t=timestamp,v1=hex` — verify both the signature and
that the timestamp is recent (to prevent replay attacks).

```typescript
function verifyStripe(rawBody: string, headers: Record<string, string>): void {
  const secret = process.env.STRIPE_WEBHOOK_SECRET;
  if (!secret) throw new WebhookVerificationError("STRIPE_WEBHOOK_SECRET not configured");

  const sigHeader = headers["stripe-signature"];
  if (!sigHeader) throw new WebhookVerificationError("Missing Stripe signature");

  const parts = Object.fromEntries(
    sigHeader.split(",").map((p) => p.split("=").map((s) => s.trim()) as [string, string]),
  );
  const timestamp = parts.t;
  const signature = parts.v1;
  if (!timestamp || !signature) throw new WebhookVerificationError("Malformed Stripe signature");

  // Reject events older than 5 minutes (replay protection)
  const ageSec = Math.abs(Date.now() / 1000 - Number(timestamp));
  if (!Number.isFinite(ageSec) || ageSec > 300) {
    throw new WebhookVerificationError("Stripe timestamp out of tolerance");
  }

  const expected = crypto
    .createHmac("sha256", secret)
    .update(`${timestamp}.${rawBody}`)
    .digest("hex");

  if (signature.length !== expected.length) {
    throw new WebhookVerificationError("Invalid Stripe signature");
  }
  if (!crypto.timingSafeEqual(Buffer.from(signature, "hex"), Buffer.from(expected, "hex"))) {
    throw new WebhookVerificationError("Invalid Stripe signature");
  }
}
```

### Generic HMAC-SHA256 (header name varies)

For providers using a simple `hex(HMAC-SHA256(secret, rawBody))` scheme:

```typescript
function verifyHmac(
  rawBody: string,
  headers: Record<string, string>,
  headerName: string,
  secretEnv: string,
): void {
  const secret = process.env[secretEnv];
  if (!secret) throw new WebhookVerificationError(`${secretEnv} not configured`);

  const signature = headers[headerName.toLowerCase()];
  if (!signature) throw new WebhookVerificationError("Missing signature header");

  const expected = crypto.createHmac("sha256", secret).update(rawBody).digest("hex");

  if (signature.length !== expected.length) {
    throw new WebhookVerificationError("Invalid signature");
  }
  if (!crypto.timingSafeEqual(Buffer.from(signature, "hex"), Buffer.from(expected, "hex"))) {
    throw new WebhookVerificationError("Invalid signature");
  }
}
```

---

## Retries and Failure Handling

When a request reaches Notion, Notion validates the URL, enqueues the event,
and immediately returns `202 Accepted` to the sender. Your handler runs
asynchronously after that.

| Outcome | Notion's behaviour |
|---------|-------------------|
| Handler succeeds | Run is logged. Consecutive verification-failure counter resets. |
| Handler throws `WebhookVerificationError` | Run logged as a verification failure. **Not retried.** 5 in a row → webhook blocked until next deploy. |
| Handler throws any other error | **Retried up to 3 times.** |
| Handler runs past timeout | Treated as failure; retried. |

Because the HTTP 202 is returned before your handler runs, the sender cannot
tell whether your processing succeeded. Don't rely on the provider's own retry
logic to mask Worker failures — fix them and use the retries Notion gives you.

---

## Idempotency

Workers retry, providers retry, and the same logical event can arrive multiple
times. Plan for it:

1. **Prefer the provider's event ID over `deliveryId`** for deduplication.
   `deliveryId` is stable across Notion-side retries, but a provider that
   resends an event will get a new `deliveryId`.
2. **Read the upstream record before writing** to Notion. If the page already
   reflects this event, no-op.
3. **Use the upstream record's stable ID as the Notion page's external key.**
   Querying by that property before creating a page prevents duplicate creates.
4. **Side effects** (sending an email, creating a Linear issue) need their own
   idempotency keys — most APIs accept one.

```typescript
worker.webhook("onCalendlyBooking", {
  title: "Calendly Booking",
  description: "Creates a Notion page when a Calendly meeting is booked.",
  execute: async (events, { notion }) => {
    const databaseId = process.env.MEETINGS_DB_ID;
    if (!databaseId) throw new Error("MEETINGS_DB_ID not configured");

    for (const event of events) {
      const uri = typeof event.body.payload === "object" && event.body.payload
        ? (event.body.payload as Record<string, unknown>).uri
        : undefined;
      if (typeof uri !== "string") continue;

      // Skip if we've already recorded this event
      const existing = await notion.databases.query({
        database_id: databaseId,
        filter: { property: "Event URI", rich_text: { equals: uri } },
        page_size: 1,
      });
      if (existing.results.length > 0) continue;

      await notion.pages.create({
        parent: { database_id: databaseId },
        properties: {
          Name: { title: [{ text: { content: `Calendly: ${uri}` } }] },
          "Event URI": { rich_text: [{ text: { content: uri } }] },
        },
      });
    }
  },
});
```

---

## Using Notion from a Webhook

Webhook handlers receive `context.notion`, but unlike tools called by Custom
Agents, the client is **not** authenticated automatically. You must provide
`NOTION_API_TOKEN`:

```bash
ntn workers env set NOTION_API_TOKEN=ntn_...
```

Two token options:

- **Personal access token** — acts as you. Has access to everything you can see
  in Notion. Simpler.
- **Internal integration** — acts as a bot. Must be explicitly connected to
  each page/database the Worker needs (via the page's three-dot menu →
  Connections).

```typescript
worker.webhook("createPageFromWebhook", {
  title: "Create Page From Webhook",
  description: "Creates a page when an external event is received.",
  execute: async (events, { notion }) => {
    const databaseId = process.env.WEBHOOK_DB_ID;
    if (!databaseId) throw new Error("WEBHOOK_DB_ID not configured");

    for (const event of events) {
      const externalId = typeof event.body.id === "string"
        ? event.body.id
        : event.deliveryId;

      await notion.pages.create({
        parent: { database_id: databaseId },
        properties: {
          Name: {
            title: [{ text: { content: `Webhook event ${externalId}` } }],
          },
        },
      });
    }
  },
});
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Webhook never fires | Provider not configured with the URL, or sending to wrong path | `ntn workers webhooks list` to confirm URL; check provider's webhook log |
| Provider reports 4xx/5xx from Notion | URL is wrong, or Notion can't accept the request shape | Confirm URL exactly; ensure provider sends `POST` |
| Handler runs but never finishes | Hit per-execution timeout | Move long-running work to a sync, or split into smaller chunks |
| Signature verification always fails | Computed on parsed `body`, not `rawBody` | Use `event.rawBody` |
| Signature fails after deploy | Secret value not set in remote env | `ntn workers env set ...`; verify with `ntn workers env list` |
| Webhook blocked, won't receive events | 5 consecutive `WebhookVerificationError` | Fix verification logic, then redeploy to unblock |
| Same event processed twice | Provider redelivered after timeout; or 3-retry kicked in | Add idempotency check using provider's event ID, not `deliveryId` |
| Notion API calls in handler return 401/403 | `NOTION_API_TOKEN` not set, or integration not connected to target page | Set the secret; for internal integrations, connect them to each page |
| Can't find the run in logs | Logs are per-run; need the runId | `ntn workers runs list` to find recent runs |
