# Sync Patterns Reference

Detailed guidance on building Worker syncs — scheduled jobs that pull external
data into Notion databases.

---

## Table of Contents

1. [Sync Modes](#sync-modes)
2. [Incremental Sync with Cursors](#incremental-sync-with-cursors)
3. [Batch Pagination](#batch-pagination)
4. [Backfill + Delta Pattern](#backfill--delta-pattern)
5. [Schema Design](#schema-design)
6. [Error Handling](#error-handling)
7. [Troubleshooting](#troubleshooting)

---

## Sync Modes

### Replace Mode

Overwrites the entire dataset each run. Simple but expensive for large datasets.
Best for small, frequently-changing datasets where you want a clean snapshot.

```typescript
worker.sync("dailyReport", {
  title: "Daily Sales Report",
  description: "Full refresh of today's sales data",
  schedule: "1d",
  mode: "replace",
  schema: {
    properties: [
      { name: "Order ID", type: "title" },
      { name: "Customer", type: "rich_text" },
      { name: "Amount", type: "number" },
      { name: "Date", type: "date" },
    ],
  },
  execute: async () => {
    const data = await fetchTodaysSales();
    return {
      changes: data.map(row => ({
        id: row.orderId,
        properties: {
          "Order ID": row.orderId,
          "Customer": row.customer,
          "Amount": row.amount,
          "Date": row.date,
        },
      })),
      hasMore: false,
      nextState: {},
    };
  },
});
```

### Incremental Mode

Uses cursor state to fetch only new or changed records since the last run. More
efficient for large, append-heavy datasets.

```typescript
worker.sync("newOrders", {
  title: "New Orders Sync",
  description: "Incrementally syncs new orders since last check",
  schedule: "15m",
  mode: "incremental",
  schema: {
    properties: [
      { name: "Order ID", type: "title" },
      { name: "Customer", type: "rich_text" },
      { name: "Amount", type: "number" },
      { name: "Created At", type: "date" },
    ],
  },
  execute: async ({ state }) => {
    const since = state?.lastSync ?? new Date(0).toISOString();
    const data = await fetchOrdersSince(since);

    const now = new Date().toISOString();
    return {
      changes: data.map(row => ({
        id: row.orderId,
        properties: {
          "Order ID": row.orderId,
          "Customer": row.customer,
          "Amount": row.amount,
          "Created At": row.createdAt,
        },
      })),
      hasMore: false,
      nextState: { ...state, lastSync: now },
    };
  },
});
```

---

## Incremental Sync with Cursors

The `state` object persists between runs. Use it to track:

- **Timestamp cursors**: `lastSync` ISO date string — fetch records modified
  after this time
- **Page tokens**: API-provided cursor for resuming pagination
- **Offset counters**: For APIs that use offset-based pagination

```typescript
execute: async ({ state }) => {
  const cursor = state?.cursor ?? null;
  const response = await fetch(
    `https://api.example.com/records?cursor=${cursor ?? ""}&limit=100`
  );
  const data = await response.json();

  return {
    changes: data.results.map(transformRecord),
    hasMore: data.hasMore,
    nextState: {
      ...state,
      cursor: data.nextCursor,
    },
  };
}
```

**Important**: Deploying does NOT reset sync state. The sync resumes from its
last cursor position. To restart from scratch:

```bash
ntn workers sync state reset <syncKey>
```

---

## Batch Pagination

For APIs that return large result sets, paginate within a single sync run using
`hasMore` and `nextState`. The platform calls `execute` again with the updated
state.

Keep batches around 100 records to stay within the timeout window.

```typescript
execute: async ({ state }) => {
  const page = state?.page ?? 1;
  const response = await fetch(
    `https://api.example.com/data?page=${page}&per_page=100`
  );
  const data = await response.json();

  return {
    changes: data.items.map(item => ({
      id: item.id,
      properties: { /* ... */ },
    })),
    hasMore: data.totalPages > page,
    nextState: { ...state, page: page + 1 },
  };
}
```

---

## Backfill + Delta Pattern

Notion recommends this for production use: pair a manual full-refresh sync with
a frequent incremental sync.

**Full backfill** (manual trigger, replace mode):
```typescript
worker.sync("fullBackfill", {
  title: "Full Data Backfill",
  description: "Complete data refresh — run manually",
  schedule: "manual",
  mode: "replace",
  // ...
});
```

**Incremental delta** (frequent, incremental mode):
```typescript
worker.sync("deltaSync", {
  title: "Delta Sync",
  description: "Catch new/changed records since last run",
  schedule: "15m",
  mode: "incremental",
  // ...
});
```

Run the backfill manually when you need a clean slate:
```bash
ntn workers sync trigger fullBackfill
```

The delta sync handles everything in between.

---

## Schema Design

The sync schema defines the Notion database columns. Match these to the data
you're pulling.

### Supported property types

| Type | Notion Column Type | Value Format |
|------|-------------------|--------------|
| `title` | Title | String |
| `rich_text` | Text | String |
| `number` | Number | Number |
| `select` | Select | String (must match an option) |
| `multi_select` | Multi-select | Array of strings |
| `date` | Date | ISO 8601 string |
| `checkbox` | Checkbox | Boolean |
| `url` | URL | String (valid URL) |
| `email` | Email | String (valid email) |
| `phone_number` | Phone | String |

### Design considerations

- **Every sync needs exactly one `title` property.** This is the page name in
  Notion.
- **Use `id` in each change record** for upsert matching. The platform uses
  this to update existing rows rather than creating duplicates.
- **Select options must be pre-declared** in the schema if you know them. For
  dynamic values, use `rich_text` instead.
- **Keep schemas stable.** Changing the schema after initial sync may require
  a state reset and redeployment. Plan the schema carefully before deploying.
- **Column renames in the source** won't automatically update in the Worker
  code — you'll get empty values or missed columns until you update the code
  and redeploy.

---

## Error Handling

Never throw raw errors. Return structured responses so the platform can log
and report correctly.

```typescript
execute: async ({ state }) => {
  try {
    const response = await fetch("https://api.example.com/data", {
      headers: { Authorization: `Bearer ${process.env.API_KEY}` },
    });

    if (!response.ok) {
      return {
        changes: [],
        hasMore: false,
        nextState: state,  // Preserve state so next run can retry
        error: `API returned ${response.status}: ${response.statusText}`,
      };
    }

    const data = await response.json();
    return {
      changes: data.map(transformRecord),
      hasMore: false,
      nextState: { ...state, lastSync: new Date().toISOString() },
    };
  } catch (err) {
    return {
      changes: [],
      hasMore: false,
      nextState: state,
      error: `Fetch failed: ${err.message}`,
    };
  }
}
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Sync stuck in INITIALIZING | First run hasn't completed or timed out | Check logs: `ntn workers runs logs <runId>` |
| Sync in ERROR state | 3+ consecutive failures | Check logs, fix the issue, redeploy. State is preserved. |
| Duplicate rows appearing | Missing or inconsistent `id` in change records | Ensure every record has a stable, unique `id` from the source |
| Empty values in some columns | Schema mismatch between code and source data | Verify property names match exactly (case-sensitive) |
| Timeout errors | Too much data in a single execution | Reduce batch size, use `hasMore` pagination |
| OAuth token expired | Refresh flow failed | Re-run `ntn workers oauth start <oauthName>` |
| Data not updating after redeploy | Sync resumes from last cursor, not from scratch | Run `ntn workers sync state reset <key>` if you need a fresh start |
| Sync paused unexpectedly | Manually disabled or hit error threshold | Check `ntn workers capabilities list` and re-enable if needed |
