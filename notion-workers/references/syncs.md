# Sync Patterns Reference

Detailed guidance on building Worker syncs — scheduled jobs that pull external
data into Notion databases managed by the Worker.

---

## Table of Contents

1. [Migration notes (April 2026 → current)](#migration-notes)
2. [Architecture: database + sync split](#architecture-database--sync-split)
3. [Sync Modes](#sync-modes)
4. [Incremental Sync with Cursors](#incremental-sync-with-cursors)
5. [Batch Pagination](#batch-pagination)
6. [Backfill + Delta Pattern](#backfill--delta-pattern)
7. [Cross-Database Relations](#cross-database-relations)
8. [Schema and Builder Reference](#schema-and-builder-reference)
9. [Rate Limiting with Pacers](#rate-limiting-with-pacers)
10. [Error Handling](#error-handling)
11. [Troubleshooting](#troubleshooting)

---

## Migration notes

If you're updating a Worker built against the earlier sync API (April 2026 or
older), here's what changed:

- **Schema moved out of `worker.sync()`.** Declare the database with
  `worker.database()` first, then attach syncs to it. The old
  `schema: { properties: [{ name, type }] }` inline shape is gone.
- **Three separate schema imports.** `j` from `@notionhq/workers/schema-builder`
  is for tool I/O only. Database property *definitions* now come from `Schema`
  (`@notionhq/workers/schema`), and property *values* in sync changes come
  from `Builder` (`@notionhq/workers/builder`).
- **Change records use `type` and `key`.** Each change is
  `{ type: "upsert" | "delete", key, properties }`. The old `{ id, properties }`
  shape and the use of `id` as the match field are gone — `key` matches the
  value of `primaryKeyProperty` on the database.
- **Replace mode does mark-and-sweep deletes.** Rows not seen by the time
  `hasMore: false` returns are deleted automatically. The old "you must
  explicitly delete everything" pattern is gone.
- **Property values are tokens, not strings.** Set `Status` with
  `Builder.select(item.status)`, not `item.status`. Same for dates, numbers,
  emails, etc.
- **Schedule minimum bumped to `5m`.** `1m` is no longer supported.
- **Relations between Worker-managed databases** are now native via
  `Schema.relation(databaseKey, { twoWay, relatedPropertyName })`.

If you have a working sync on the old API, expect to rewrite the database
declaration, the sync's `execute` return shape, and all property-value
construction. The execute loop structure (`state`, `hasMore`, `nextState`) is
unchanged.

---

## Architecture: database + sync split

The new pattern is two steps. Declare the database first; it returns an opaque
handle. Then register one or more syncs that target that handle.

```typescript
import { Worker } from "@notionhq/workers";
import * as Schema from "@notionhq/workers/schema";
import * as Builder from "@notionhq/workers/builder";

const worker = new Worker();
export default worker;

// 1. Declare the managed database
const issues = worker.database("issues", {
  type: "managed",
  initialTitle: "Issues",
  primaryKeyProperty: "Issue ID",
  schema: {
    properties: {
      Name: Schema.title(),
      "Issue ID": Schema.richText(),
      Status: Schema.select([
        { name: "Open" },
        { name: "Closed", color: "green" },
      ]),
      Priority: Schema.select([
        { name: "Low" },
        { name: "Medium" },
        { name: "High", color: "red" },
      ]),
    },
  },
});

// 2. Register a sync against it
worker.sync("issuesSync", {
  database: issues,
  mode: "replace",
  schedule: "1h",
  execute: async (state) => {
    const page = state?.page ?? 1;
    const { items, hasMore } = await fetchIssues(page, 100);
    return {
      changes: items.map((item) => ({
        type: "upsert" as const,
        key: item.id,
        properties: {
          Name: Builder.title(item.title),
          "Issue ID": Builder.richText(item.id),
          Status: Builder.select(item.status),
          Priority: Builder.select(item.priority),
        },
      })),
      hasMore,
      nextState: hasMore ? { page: page + 1 } : undefined,
    };
  },
});
```

A few things to note:

- `primaryKeyProperty` must reference a property declared in `schema.properties`.
  It's typically the upstream API's ID column.
- The `key` field on each change record matches the value Notion stores in the
  primary key property. Use the upstream record's ID.
- `worker.database()` returns an opaque handle — pass it into `worker.sync()`'s
  `database` field, don't try to inspect it.
- Notion creates and migrates the managed database on every deploy. Schema
  changes can drop data, so review before deploying.
- Multiple syncs can target the same database — see the backfill + delta
  pattern below.

---

## Sync Modes

### Replace Mode (default)

Every sync cycle returns the **full upstream dataset**. After the final
`hasMore: false`, Notion deletes any rows that weren't seen during the cycle
(mark-and-sweep). Best for small datasets (under ~10k records) or APIs that
don't expose a change feed.

```typescript
worker.sync("teamsSync", {
  database: teams,
  mode: "replace",
  schedule: "1d",
  execute: async (state) => {
    const page = state?.page ?? 1;
    const { items, hasMore } = await fetchTeams(page, 100);
    return {
      changes: items.map((item) => ({
        type: "upsert" as const,
        key: item.id,
        properties: {
          Name: Builder.title(item.name),
          ID: Builder.richText(item.id),
        },
      })),
      hasMore,
      nextState: hasMore ? { page: page + 1 } : undefined,
    };
  },
});
```

If your sync errors partway through a cycle, no deletions happen for that
cycle — this is intentional to avoid data loss.

### Incremental Mode

Every cycle returns only **changes since the last cursor**. Rows not mentioned
are left alone. Deletions must be explicit. Best for large datasets where the
upstream API exposes change tracking.

```typescript
worker.sync("eventsSync", {
  database: events,
  mode: "incremental",
  schedule: "5m",
  execute: async (state) => {
    const { upserts, deletes, nextCursor } = await fetchChanges(state?.cursor);
    return {
      changes: [
        ...upserts.map((item) => ({
          type: "upsert" as const,
          key: item.id,
          properties: {
            Name: Builder.title(item.name),
            ID: Builder.richText(item.id),
          },
        })),
        ...deletes.map((id) => ({
          type: "delete" as const,
          key: id,
        })),
      ],
      hasMore: Boolean(nextCursor),
      nextState: nextCursor ? { cursor: nextCursor } : undefined,
    };
  },
});
```

---

## Incremental Sync with Cursors

The `state` argument persists between executions and between cycles. Use it for:

- **Timestamp cursors** (`lastSync` as ISO string) for "modified after" queries
- **Page tokens** from the upstream API
- **Offset counters** for offset/limit pagination

```typescript
execute: async (state) => {
  const cursor = state?.cursor ?? null;
  await api.wait();
  const response = await fetch(
    `https://api.example.com/records?cursor=${cursor ?? ""}&limit=100`
  );
  const data = await response.json();

  return {
    changes: data.results.map(toUpsert),
    hasMore: Boolean(data.nextCursor),
    nextState: data.nextCursor ? { cursor: data.nextCursor } : undefined,
  };
}
```

**Eventual consistency tip**: For APIs that aren't strictly ordered, keep the
timestamp cursor slightly behind "now" so recently written records aren't
skipped:

```typescript
const bufferedNow = new Date(Date.now() - 15_000).toISOString();
const latestReturned = records.at(-1)?.updatedAt;
const cursor = latestReturned && latestReturned < bufferedNow
  ? latestReturned
  : bufferedNow;

return {
  changes: records.map(toUpsert),
  hasMore: false,
  nextState: { cursor },
};
```

**Important**: Deploying does NOT reset sync state. The sync resumes from its
last cursor. To restart:

```bash
ntn workers sync state reset <syncKey>
```

To inspect the current state before deciding:

```bash
ntn workers sync state get <syncKey>
```

---

## Batch Pagination

For large result sets, paginate within a single sync cycle using `hasMore`
and `nextState`. The runtime calls `execute` again with the new state.

Keep batches around 100 records to stay within the timeout.

```typescript
execute: async (state) => {
  const page = state?.page ?? 1;
  await api.wait();
  const { items, hasMore } = await fetchPage(page, 100);
  return {
    changes: items.map((item) => ({
      type: "upsert" as const,
      key: item.id,
      properties: { /* ... */ },
    })),
    hasMore,
    nextState: hasMore ? { page: page + 1 } : undefined,
  };
}
```

---

## Backfill + Delta Pattern

Notion's recommended production pattern: pair a manual replace-mode backfill
with a frequent incremental delta, both targeting the same database.

The delta keeps the data fresh; the backfill catches anything the delta misses
(deletes the upstream API didn't surface, schema additions, drift after bug
fixes).

```typescript
// Delta: near-real-time updates
worker.sync("ticketsDelta", {
  database: tickets,
  mode: "incremental",
  schedule: "5m",
  execute: async (state) => {
    await apiPacer.wait();
    const { items, nextCursor } = await fetchTicketChanges(state?.cursor);
    return {
      changes: items.map((t) => ({
        type: "upsert" as const,
        key: t.id,
        properties: {
          Summary: Builder.title(t.summary),
          "Ticket ID": Builder.richText(t.id),
        },
      })),
      hasMore: Boolean(nextCursor),
      nextState: nextCursor ? { cursor: nextCursor } : undefined,
    };
  },
});

// Backfill: full dataset sweep, run manually
worker.sync("ticketsBackfill", {
  database: tickets,
  mode: "replace",
  schedule: "manual",
  execute: async (state) => {
    const page = state?.page ?? 1;
    await apiPacer.wait();
    const { items, hasMore } = await fetchAllTickets(page);
    return {
      changes: items.map((t) => ({
        type: "upsert" as const,
        key: t.id,
        properties: {
          Summary: Builder.title(t.summary),
          "Ticket ID": Builder.richText(t.id),
        },
      })),
      hasMore,
      nextState: hasMore ? { page: page + 1 } : undefined,
    };
  },
});
```

To run a backfill:

```bash
ntn workers sync state reset ticketsBackfill
ntn workers sync trigger ticketsBackfill
```

If both syncs hit the same upstream API, give them the same pacer. The runtime
divides the rate-limit budget between concurrently executing capabilities.

---

## Cross-Database Relations

Use `Schema.relation()` to link two managed databases declared in the same
Worker:

```typescript
const projects = worker.database("projects", {
  type: "managed",
  initialTitle: "Projects",
  primaryKeyProperty: "Project ID",
  schema: {
    properties: {
      Name: Schema.title(),
      "Project ID": Schema.richText(),
    },
  },
});

const tasks = worker.database("tasks", {
  type: "managed",
  initialTitle: "Tasks",
  primaryKeyProperty: "Task ID",
  schema: {
    properties: {
      Name: Schema.title(),
      "Task ID": Schema.richText(),
      Project: Schema.relation("projects", {
        twoWay: true,
        relatedPropertyName: "Tasks",   // adds a "Tasks" rollup on Projects
      }),
    },
  },
});

worker.sync("tasksSync", {
  database: tasks,
  execute: async () => {
    const items = await fetchTasks();
    return {
      changes: items.map((task) => ({
        type: "upsert" as const,
        key: task.id,
        properties: {
          Name: Builder.title(task.name),
          "Task ID": Builder.richText(task.id),
          Project: [Builder.relation(task.projectId)],   // array of references
        },
      })),
      hasMore: false,
    };
  },
});
```

The first argument to `Schema.relation()` is the key passed to the related
`worker.database()` (the string `"projects"`, not the JS variable name).

Relation property values are arrays of `Builder.relation(primaryKey)` calls.
Multiple relations:

```typescript
Projects: [
  Builder.relation("project-123"),
  Builder.relation("project-456"),
]
```

---

## Schema and Builder Reference

### Database property definitions (`Schema`)

```typescript
import * as Schema from "@notionhq/workers/schema";

Schema.title()                 // exactly one per database
Schema.richText()              // plain text column
Schema.url()
Schema.email()
Schema.phoneNumber()
Schema.checkbox()
Schema.file()
Schema.number()                // or Schema.number("dollar") for currency format
Schema.date()                  // or Schema.date("YYYY/MM/DD")
Schema.select([{ name: "Open" }, { name: "Done", color: "green" }])
Schema.multiSelect([{ name: "Bug", color: "red" }])
Schema.status({
  groups: [
    { name: "To-do", options: [{ name: "Not started" }] },
    { name: "Complete", options: [{ name: "Done", color: "green" }] },
  ],
})
Schema.people()
Schema.place()
Schema.relation("databaseKey")
Schema.relation("databaseKey", { twoWay: true, relatedPropertyName: "Tasks" })
```

### Property values in sync changes (`Builder`)

```typescript
import * as Builder from "@notionhq/workers/builder";

Builder.title("Page name")
Builder.richText("Some text")
Builder.text("Same as richText")
Builder.url("https://example.com")
Builder.email("a@example.com")
Builder.phoneNumber("+14155550123")
Builder.checkbox(true)
Builder.file("https://example.com/x.pdf", "Invoice")
Builder.number(42)
Builder.date("2026-05-11")                         // YYYY-MM-DD
Builder.dateTime("2026-05-11T09:30:00Z")           // ISO 8601, optional timezone
Builder.dateTime("2026-05-11T09:30:00Z", "Asia/Singapore")
Builder.dateRange("2026-05-11", "2026-05-15")
Builder.dateTimeRange("2026-05-11T09:30:00Z", "2026-05-11T10:30:00Z")
Builder.select("Open")
Builder.multiSelect("Bug", "Customer")
Builder.status("Done")
Builder.people("a@example.com", "b@example.com")
Builder.place({ lat: 1.3521, lon: 103.8198, name: "Singapore" })
Builder.link("Display text", "https://example.com")

// Icons (for an upsert change's `icon` field)
Builder.emojiIcon("✅")
Builder.notionIcon("checkmark", "green")
Builder.imageIcon("https://example.com/icon.png")
```

A change record can also carry optional fields:

```typescript
{
  type: "upsert",
  key: "task-123",
  properties: { /* ... */ },
  upstreamUpdatedAt: "2026-05-11T09:30:00Z",   // for conflict resolution between syncs
  icon: Builder.emojiIcon("📝"),
  pageContentMarkdown: "Imported from the upstream task tracker.",
}
```

For the complete list of helpers, see
https://developers.notion.com/workers/reference/schema.

---

## Rate Limiting with Pacers

A pacer spreads outbound API calls across a time window so the Worker doesn't
burn through the upstream rate limit immediately.

```typescript
const api = worker.pacer("api", {
  allowedRequests: 10,
  intervalMs: 1000,           // 10 req/sec
});

worker.sync("customersSync", {
  database: customers,
  execute: async (state) => {
    await api.wait();           // blocks until a slot is available
    const data = await fetchCustomers(state?.cursor);
    // ...
  },
});
```

When multiple capabilities share a pacer, the platform divides the budget
across concurrently executing capabilities. Use the same pacer for delta and
backfill syncs that hit the same upstream API.

---

## Error Handling

Don't throw raw errors from sync `execute`. Return structured results so the
platform can log correctly and your state is preserved for the next retry.

```typescript
execute: async (state) => {
  try {
    await api.wait();
    const response = await fetch("https://api.example.com/data", {
      headers: { Authorization: `Bearer ${process.env.API_KEY}` },
    });

    if (!response.ok) {
      return {
        changes: [],
        hasMore: false,
        nextState: state,                       // preserve state for retry
        error: `API returned ${response.status}: ${response.statusText}`,
      };
    }

    const data = await response.json();
    return {
      changes: data.map(toUpsert),
      hasMore: false,
      nextState: { ...state, lastSync: new Date().toISOString() },
    };
  } catch (err) {
    return {
      changes: [],
      hasMore: false,
      nextState: state,
      error: `Fetch failed: ${(err as Error).message}`,
    };
  }
}
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Sync runs but no rows appear | Data-fetch returning empty, or `key` doesn't match `primaryKeyProperty` | Run `ntn workers sync trigger <key> --preview` to inspect what `execute` returns |
| Rows are duplicated | Inconsistent `key` between cycles | Ensure `key` is the stable upstream ID, not a value that changes between runs |
| Stale rows aren't deleted (replace mode) | Sync errored partway through, so mark-and-sweep didn't run | Fix the error so the cycle reaches `hasMore: false` |
| Sync stuck in INITIALIZING | First run hasn't completed | Check logs: `ntn workers runs list` then `ntn workers runs logs <runId>` |
| Sync in ERROR state | 3+ consecutive failures | Check logs, fix the issue, redeploy. State is preserved. |
| Empty values in some columns | Property name mismatch between code and schema | Property names are case-sensitive; check exact match |
| Timeout errors | Too much data in a single execution | Reduce batch size, paginate with `hasMore` / `nextState` |
| OAuth token expired | Refresh flow failed | Re-run `ntn workers oauth start <capabilityKey>` |
| Data didn't update after redeploy | Sync resumed from old cursor | Run `ntn workers sync state reset <syncKey>` if you need a fresh start |
| Sync paused unexpectedly | Manually disabled or hit error threshold | `ntn workers capabilities list`, then `enable <key>` if needed |
