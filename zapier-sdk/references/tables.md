# Zapier Tables Reference

Zapier Tables is a structured data store accessible via the SDK and CLI. Think of it as a lightweight database you can create, populate, and query programmatically.

## CLI Usage

### Create a table
```bash
npx zapier-sdk create-table "Leads" --description "Sales leads tracker" --json
```

### Add fields (columns)
```bash
npx zapier-sdk create-table-fields TABLE_ID '[
  {"name": "Company", "type": "text"},
  {"name": "Status", "type": "text"},
  {"name": "Revenue", "type": "number"}
]'
```

### Create records (max 100 per call)
```bash
npx zapier-sdk create-table-records TABLE_ID '[
  {"Company": "Acme", "Status": "Active", "Revenue": 50000},
  {"Company": "Globex", "Status": "Lead", "Revenue": 120000}
]' --key-mode names
```

### Query records
```bash
npx zapier-sdk list-table-records TABLE_ID \
  --filters '[{"fieldKey":"Status","operator":"equals","value":"Active"}]' \
  --sort '{"fieldKey":"Revenue","direction":"desc"}' \
  --json
```

`--filters` is an **array** of `{fieldKey, operator, value}` conditions. `--sort` is a single `{fieldKey, direction}` object. The older `--filters '{"Status":"Active"}' --sort "Revenue" --direction desc` shape is no longer supported.

### Update records
```bash
npx zapier-sdk update-table-records TABLE_ID '[
  {"id": "rec_abc", "Status": "Closed Won"}
]' --key-mode names
```

### Delete records
```bash
npx zapier-sdk delete-table-records TABLE_ID '["rec_abc", "rec_def"]'
```

## TypeScript SDK Usage

```typescript
import { createZapierSdk } from "@zapier/zapier-sdk";
const zapier = createZapierSdk();

// Create table
const { data: table } = await zapier.createTable({
  name: "Leads",
  description: "Sales leads",
});

// Add fields
await zapier.createTableFields({
  table: table.id,
  fields: [
    { name: "Company", type: "text" },
    { name: "Status", type: "text" },
    { name: "Revenue", type: "number" },
  ],
});

// Create records (max 100 per call)
await zapier.createTableRecords({
  table: table.id,
  records: [
    { Company: "Acme", Status: "Active", Revenue: 50000 },
  ],
  keyMode: "names", // use field names (default) vs internal IDs
});

// Query records with filters and sorting
const { data: records } = await zapier.listTableRecords({
  table: table.id,
  filters: [
    { fieldKey: "Status", operator: "equals", value: "Active" },
  ],
  sort: { fieldKey: "Revenue", direction: "desc" },
});

// Update records
await zapier.updateTableRecords({
  table: table.id,
  records: [{ id: "rec_abc", Status: "Closed Won" }],
  keyMode: "names",
});

// Delete records
await zapier.deleteTableRecords({
  table: table.id,
  records: ["rec_abc", "rec_def"],
});

// List tables
const { data: tables } = await zapier.listTables({
  owner: "me",
  search: "leads",
});

// Get table details
const { data: details } = await zapier.getTable({ table: table.id });

// List fields
const { data: fields } = await zapier.listTableFields({ table: table.id });

// Delete fields
await zapier.deleteTableFields({ table: table.id, fields: ["f3", "f4"] });

// Delete table
await zapier.deleteTable({ table: table.id });
```

## Key Mode

The `keyMode` parameter controls how field references work:

- `"names"` (default): Use human-readable field names like `"Company"`, `"Status"`
- `"ids"`: Use internal field IDs like `"f1"`, `"f2"`

Use `"names"` for readability in most cases. Use `"ids"` when field names might be ambiguous or when working with programmatically generated schemas.
