# Zapier Tables Reference

Zapier Tables is Zapier's first-party structured data store — fields, records, views, and formula columns. Accessible from `@zapier/zapier-sdk` via `createZapierSdk()`. **Auth is automatic** — no connection needed, works from background agents.

```typescript
import { createZapierSdk } from "@zapier/zapier-sdk";
const zapier = createZapierSdk();
```

> **Param naming:** every Tables method takes `table: "<id>"` (suffix-less, matching `app`/`connection`/`action` elsewhere in the SDK). The old `tableId` form is deprecated but still works. Use `table` in new code.

## Two interfaces — pick the right one

| Interface | Best for | Limits |
|---|---|---|
| **Native SDK methods** (`zapier.listTableRecords()`, `createTableRecords()`, etc.) | Bulk create/update/delete (≤100/call), pagination, field CRUD, sorting, simple filtering | Field references controlled by `keyMode` |
| **Action layer** (`zapier.runAction({ appKey: "zapier-tables", ... })`) | Advanced search operators (`gt`/`range`/`icontains`), atomic increment, upsert | One record at a time; uses `f1`/`f2` IDs and `new__data__` write prefixes |

Default to the **native methods** — they're cleaner, support bulk, and use human-readable field names. Drop to the **action layer** only for the capabilities native methods lack: comparison/substring search operators, `increment_value`, and `upsert_find_record`.

---

## Native SDK methods

`keyMode` controls how field keys in record `data` are interpreted: `"names"` (default, human-readable like `"Status"`) or `"ids"` (raw `f1`, `f2`).

### Tables

```typescript
// Create — returns { data: { id, ... } }
const { data: table } = await zapier.createTable({ name: "Leads", description: "Sales leads" });

// List — PaginatedResult: { data, nextCursor }. Supports async iteration.
const { data: tables, nextCursor } = await zapier.listTables({ owner: "me", search: "leads" });
for await (const t of zapier.listTables().items()) { /* per-table */ }

// Get details
const { data: details } = await zapier.getTable({ table: table.id });

// Delete
await zapier.deleteTable({ table: table.id });
```

### Fields (columns)

```typescript
// List schema
const { data: fields } = await zapier.listTableFields({ table: table.id });

// Add — verified types: text, number, string (others likely work)
await zapier.createTableFields({
  table: table.id,
  fields: [{ name: "Company", type: "text" }, { name: "Revenue", type: "number" }],
});

// Delete — accepts field names or IDs ("Email", "f6", "6", or 6)
await zapier.deleteTableFields({ table: table.id, fields: ["f3", "f4"] });
```

### Records — read

```typescript
// Single record by ID
const { data: record } = await zapier.getTableRecord({ table: table.id, record: "rec_abc" });

// List with filtering + sorting (PaginatedResult)
const { data: records, nextCursor } = await zapier.listTableRecords({
  table: table.id,
  filters: [{ fieldKey: "Status", operator: "is", value: "Active" }],
  sort: { fieldKey: "Revenue", direction: "desc" },
  pageSize: 500,            // max 1000
});

// Async iteration handles pagination for you
for await (const r of zapier.listTableRecords({ table: table.id }).items()) { /* per-record */ }
```

**Filter operators (native):** `is`, `isNot`, `contains`, `doesNotContain`, `gt`, `gte`, `lt`, `lte`, `isEmpty`, `isNotEmpty`. Filters are keyed by `fieldKey`. Sort param is `sort` (a single `{ fieldKey, direction }` object) — **not** `orders`, which is silently stripped by Zod validation.

### Records — write (bulk, max 100 per call)

```typescript
// Create
await zapier.createTableRecords({
  table: table.id,
  records: [{ Company: "Acme", Status: "Active", Revenue: 50000 }],
  keyMode: "names",   // default
});

// Update — each record object carries its id
await zapier.updateTableRecords({
  table: table.id,
  records: [{ id: "rec_abc", Status: "Closed Won" }],
  keyMode: "names",
});

// Delete — array of record IDs
await zapier.deleteTableRecords({ table: table.id, records: ["rec_abc", "rec_def"] });
```

---

## Action layer (`runAction`) — advanced search, increment, upsert

Reach here only for what native methods can't do. Uses `appKey: "zapier-tables"` and field **IDs** (`f1`, `f2`). **Read** prefix `data__fN`; **write** prefix `new__data__fN` (double underscores throughout — wrong prefixes silently no-op). All write/search actions **return arrays** — access `data[0]`.

Discover field IDs:
```typescript
const { data: fields } = await zapier.runAction({
  appKey: "zapier-tables", actionType: "read", actionKey: "list_fields_including_id",
  inputs: { table_id: TABLE_ID },
}); // → [{ id: 1, name: "title", curly_key: "data__f1" }, ...]
```

### Search with comparison/substring operators

Native filters only do equality-ish checks. For `gt`/`range`/`icontains`/etc., use the action layer:

```typescript
const { data } = await zapier.runAction({
  appKey: "zapier-tables", actionType: "search", actionKey: "find_records",
  inputs: { table_id: TABLE_ID, field_data_key: "data__f4", operator: "gt", lookup_value: "3" },
});
// find_records response is triple-nested: data[0].data[0].data.f4
```

**Operators:** `exact` (default), `different`, `contains`, `icontains`, `in`, `isnull`, `startswith`, `gt`, `gte`, `lt`, `lte`, `range`.
- `gt`/`gte`/`lt`/`lte`/`range` work on **numeric** fields only (throw on text).
- `range` needs `lookup_value` (min) + `lookup_value2` (max).
- Up to 5 filters, AND-combined (no OR). Use `filter_count` + suffixed keys (`field_data_key_2`, `operator_2`, `lookup_value_2`, …).
- `find_record` (single) nests under `old.data`; caps differ from `find_records`.

> ⚠️ Action-layer list/search has silent caps (`list_records` ~100, `find_records` ~1,000). For unfiltered reads of large tables, use native `listTableRecords` with cursor/iteration instead.

### Atomic increment

```typescript
await zapier.runAction({
  appKey: "zapier-tables", actionType: "write", actionKey: "increment_value",
  inputs: { table_id: TABLE_ID, record_id: "01KN...", field_data_key: "data__f4", increment_amount: 1 },
}); // key is field_data_key, NOT field_to_increment (docs are wrong); negative to decrement
```

### Upsert (find-or-create)

```typescript
const { data } = await zapier.runAction({
  appKey: "zapier-tables", actionType: "search", actionKey: "upsert_find_record",
  inputs: { table_id: TABLE_ID, field_data_key: "data__f1", lookup_value: "My title",
            "new__data__f1": "My title", "new__data__f2": "Description" },
});
const wasCreated = data[0].old === null;
```
> ⚠️ `upsert_find_record` **only creates if missing — it does not update an existing record.** For true upsert, use native `listTableRecords`/`getTableRecord` to find, then `updateTableRecords` or `createTableRecords`.

### Action-layer field formats

- Dropdown: pass the value string (`"🔴 Open"`); reads back as `{ value, label }`.
- Link: `new__data__f9` = URL, `new__data__f9__text` = label.
- Datetime: ISO 8601 string. Number: number. Checkbox: boolean.

---

## Gotchas (verified)

- **Param is `table`**, not `tableId` (latter deprecated).
- Native record writes are bulk (≤100/call); action-layer writes are one at a time.
- Sort param is `sort` (object), not `orders` (silently stripped).
- `listTables` / `listTableRecords` / `listTableFields` return `PaginatedResult` (`{ data, nextCursor }`) — call `.items()` to iterate, don't `.map()` the result directly.
- Action layer: write prefix is `new__data__` (double underscores); wrong prefix = silent no-op.
- Action layer: all write/search actions return **arrays** — access `data[0]`.
- Action layer: `find_record` nests under `old.data`; `find_records` is triple-nested.
- `increment_value` uses `field_data_key`, not `field_to_increment`.
- `upsert_find_record` does not update — only creates if absent.
- No OR logic in filtering anywhere; action layer is AND-only, max 5 filters.
- No unique constraints / idempotency keys — retries create duplicates; dedup is client-side.
- Formula fields are read-only.

## Background-agent compatibility

Works from all contexts — interactive chat, background agents, script automation (push). No connection needed.

## JSONata querying gotchas (action-layer output)

- Records nest under `data`: `result[data.f6 != "Done"]`, not `result[f6 != "Done"]`.
- Dropdowns are objects: `result[data.f6.value != "Done"]`.
- No `(?i)` regex flags: use `result[data.f2 ~> /[Ss]tabil/]`.
