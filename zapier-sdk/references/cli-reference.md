# CLI Reference

Complete reference for `@zapier/zapier-sdk-cli` v0.39.1.

## Global Options

Available on all commands:

| Flag | Description |
|------|-------------|
| `--json` | Output as JSON (essential for scripting) |
| `--credentials` | Auth credentials string |
| `--credentials-client-id` | Client ID for auth |
| `--credentials-client-secret` | Client secret for auth |
| `--max-network-retries` | Max retry count |
| `--max-network-retry-delay-ms` | Max retry delay |
| `--debug` | Enable debug output |

---

## Account Commands

### login
Authenticate with Zapier via browser.
```bash
npx zapier-sdk login [--timeout 300]
```

### logout
End current session.
```bash
npx zapier-sdk logout
```

### get-profile
Get current user info.
```bash
npx zapier-sdk get-profile [--json]
```

### get-login-config-path
Show where auth config is stored.
```bash
npx zapier-sdk get-login-config-path
```

---

## App Commands

### list-apps
Browse available apps.
```bash
npx zapier-sdk list-apps [--search "name"] [--apps app1,app2] [--page-size N] [--max-items N] [--cursor X] [--json]
```

### get-app
Get details about a specific app. Accepts slug, implementation name, or versioned ID.
```bash
npx zapier-sdk get-app <app> [--json]
```

---

## Action Commands

### list-actions
List actions for an app.
```bash
npx zapier-sdk list-actions <app> [--action-type read|write|search] [--page-size N] [--max-items N] [--cursor X] [--json]
```

### get-action
Get details about a specific action.
```bash
npx zapier-sdk get-action <app> <action-type> <action> [--json]
```

### run-action
Execute an action.
```bash
npx zapier-sdk run-action <app> <action-type> <action> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--timeout-ms 180000] \
  [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

### list-input-fields
Get required inputs for an action. Pass `--inputs` with known values to reveal dynamic fields.
```bash
npx zapier-sdk list-input-fields <app> <action-type> <action> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

### get-input-fields-schema
Get JSON Schema for action inputs (useful for agent tool definitions).
```bash
npx zapier-sdk get-input-fields-schema <app> <action-type> <action> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--json]
```

### list-input-field-choices
Get dynamic dropdown options for a specific field.
```bash
npx zapier-sdk list-input-field-choices <app> <action-type> <action> <input-field> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--page N] [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

---

## Connection Commands

### list-connections
List available connections.
```bash
npx zapier-sdk list-connections \
  [--app slug] [--owner me] [--is-expired false] \
  [--search "text"] [--title "name"] \
  [--connections id1,id2] [--account X] \
  [--include-shared] \
  [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

### find-first-connection
Get the first matching connection.
```bash
npx zapier-sdk find-first-connection \
  [--app slug] [--owner me] [--is-expired false] \
  [--search "text"] [--title "name"] \
  [--account X] [--include-shared] \
  [--json]
```

### find-unique-connection
Get exactly one matching connection (errors if zero or multiple).
```bash
npx zapier-sdk find-unique-connection \
  [--app slug] [--owner me] [--is-expired false] \
  [--json]
```

### get-connection
Get details about a specific connection.
```bash
npx zapier-sdk get-connection [--connection ID] [--json]
```

---

## HTTP / curl Command

Make authenticated HTTP requests directly. Connection ID auto-injects credentials.

```bash
npx zapier-sdk curl <url> \
  [--connection ID] \
  [-X METHOD] \
  [-H "Header: value"] \
  [--json '{"body": "data"}'] \
  [--data "raw-data"] \
  [--data-raw "data"] \
  [--data-binary @file] \
  [-F "field=value"] \
  [--get] [--head] \
  [-L] [-i] [-o file] [-v] [-s] [-f] \
  [--max-time seconds] \
  [--user user:pass]
```

Supports familiar curl-style flags. The `--connection` flag is the key differentiator — it handles OAuth token injection automatically.

---

## Table Commands

### create-table
```bash
npx zapier-sdk create-table <name> [--description "text"] [--json]
```

### list-tables
```bash
npx zapier-sdk list-tables [--kind X] [--search "text"] [--owner me] [--include-shared] [--json]
```

### get-table / delete-table
```bash
npx zapier-sdk get-table <table> [--json]
npx zapier-sdk delete-table <table>
```

### Field operations
```bash
npx zapier-sdk create-table-fields <table> <fields-json>
npx zapier-sdk list-table-fields <table> [--fields f1,f2] [--json]
npx zapier-sdk delete-table-fields <table> <fields>
```

### Record operations
```bash
npx zapier-sdk create-table-records <table> <records-json> [--key-mode names|ids]
npx zapier-sdk get-table-record <table> <record> [--key-mode names|ids] [--json]
npx zapier-sdk list-table-records <table> <field-key> [--filters X] [--sort X] [--direction asc|desc] [--key-mode names|ids] [--json]
npx zapier-sdk update-table-records <table> <records-json> [--key-mode names|ids]
npx zapier-sdk delete-table-records <table> <records>
```

Max 100 records per create/update call. `--key-mode` controls whether field references use human-readable names (default) or internal IDs (f1, f2, etc.).

---

## Development Commands

### add
Register apps and generate TypeScript types.
```bash
npx zapier-sdk add <apps> [--connections X] [--config-path X] [--types-output X]
```

### generate-app-types
Create TypeScript type definitions for specific apps.
```bash
npx zapier-sdk generate-app-types <apps> [--connections X] [--types-output-directory X]
```

### build-manifest
Compile manifest entries for apps.
```bash
npx zapier-sdk build-manifest <apps> [--skip-write] [--config-path X]
```

### bundle-code
Convert TypeScript to executable JavaScript.
```bash
npx zapier-sdk bundle-code <input> [--output X] [--string] [--minify] [--target X] [--cjs]
```

### init
Bootstrap a new SDK project.
```bash
npx zapier-sdk init <project-name> [--skip-prompts]
```

### mcp
Launch Model Context Protocol server.
```bash
npx zapier-sdk mcp [--port N]
```

### feedback
Submit feedback to Zapier.
```bash
npx zapier-sdk feedback "your feedback here"
```

---

## Client Credential Commands

### create-client-credentials
Generate credentials for programmatic access. Save the secret immediately — shown only once.
```bash
npx zapier-sdk create-client-credentials <name> [--allowed-scopes X]
```

### list-client-credentials
```bash
npx zapier-sdk list-client-credentials [--page-size N] [--max-items N] [--cursor X] [--json]
```

### delete-client-credentials
```bash
npx zapier-sdk delete-client-credentials <client-id>
```
