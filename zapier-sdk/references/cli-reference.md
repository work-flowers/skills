# CLI Reference

Complete reference for `@zapier/zapier-sdk-cli` v0.39.1.

## Global Options

Available on all commands:

| Flag | Description |
|------|-------------|
| `--json` | Output as JSON (essential for scripting) |
| `--credentials` | Auth token (single string) |
| `--credentials-client-id` | OAuth client ID |
| `--credentials-client-secret` | OAuth client secret |
| `--credentials-base-url` | Override auth base URL |
| `--base-url` | Override Zapier API base URL |
| `--tracking-base-url` | Override Zapier tracking endpoints |
| `--max-network-retries` | Max retries for rate-limited requests (default 3) |
| `--max-network-retry-delay-ms` | Max delay in ms between retries (default 60000) |
| `--approval-mode` | `disabled` (default), `poll`, or `throw` — see Approval Flow below |
| `--approval-timeout-ms` | Approval polling timeout in ms (default 600000 / 10 min) |
| `--max-approval-retries` | Max sequential approval rounds per request (default 2) |
| `--can-include-shared-connections` | Allow listing shared connections |
| `--can-include-shared-tables` | Allow listing shared tables |
| `--can-delete-tables` | Allow deleting tables |
| `--debug` | Enable debug output |

### Approval Flow

When org policy gates an action, the CLI's default behaviour is to throw `ZapierApprovalError` rather than create an approval. Override with:

- `--approval-mode poll` — create the approval, open it in a browser, poll until resolved, then retry the original request.
- `--approval-mode throw` — create the approval and throw `ZapierApprovalError` with the approval URL so the caller can surface it (e.g. in a chat UI).

You can also set the `ZAPIER_APPROVAL_MODE` env var.

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

### list-action-input-fields
Get required inputs for an action. Pass `--inputs` with known values to reveal dynamic fields.
```bash
npx zapier-sdk list-action-input-fields <app> <action-type> <action> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

### get-action-input-fields-schema
Get JSON Schema for action inputs (useful for agent tool definitions).
```bash
npx zapier-sdk get-action-input-fields-schema <app> <action-type> <action> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--json]
```

### list-action-input-field-choices
Get dynamic dropdown options for a specific field.
```bash
npx zapier-sdk list-action-input-field-choices <app> <action-type> <action> <input-field> \
  [--connection ID] \
  [--inputs '{"key": "value"}'] \
  [--page N] [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

Note: the input-field commands all carry the `action-` prefix. The shorter names (`list-input-fields`, `get-input-fields-schema`, `list-input-field-choices`) do not exist as CLI commands.

---

## Connection Commands

In all three of `list-connections`, `find-first-connection`, and `find-unique-connection`, `[app]` is an **optional positional argument**, not a flag. Non-expired connections are returned by default; pass `--expired` to filter to expired-only.

### list-connections
List available connections.
```bash
npx zapier-sdk list-connections [app] \
  [--owner me] [--expired] \
  [--search "text"] [--title "name"] \
  [--connections id1,id2] [--account X] \
  [--include-shared] \
  [--page-size N] [--max-items N] [--cursor X] \
  [--json]
```

### find-first-connection
Get the first matching connection.
```bash
npx zapier-sdk find-first-connection [app] \
  [--owner me] [--expired] \
  [--search "text"] [--title "name"] \
  [--account X] [--include-shared] \
  [--json]
```

### find-unique-connection
Get exactly one matching connection (errors if zero or multiple).
```bash
npx zapier-sdk find-unique-connection [app] \
  [--owner me] [--expired] \
  [--json]
```

### get-connection
Get details about a specific connection.
```bash
npx zapier-sdk get-connection [--connection ID] [--json]
```

---

## HTTP / curl Command

Make authenticated HTTP requests directly. The `--connection` flag auto-injects stored credentials.

```bash
npx zapier-sdk curl <url> \
  [--connection ID] \
  [--request METHOD] [-X METHOD] \
  [--header "Header: value"] [-H "Header: value"] \
  [--json '{"body": "data"}'] \
  [--data raw] [--data-raw raw] [--data-binary @file] [--data-urlencode "k=v"] \
  [--form "field=value"] [--form-string "field=val"] \
  [--get] [--head] \
  [--location] [-L] \
  [--include] [-i] \
  [--output file] [-o file] [--remote-name] \
  [--verbose] [-v] [--silent] [-s] [--show-error] \
  [--fail] [-f] [--fail-with-body] \
  [--write-out '%{http_code}'] \
  [--max-time seconds] \
  [--user user:pass] \
  [--compressed]
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
npx zapier-sdk list-table-records <table> \
  [--filters '[{"fieldKey":"f1","operator":"is","value":"active"}]'] \
  [--sort '{"fieldKey":"f1","direction":"desc"}'] \
  [--key-mode names|ids] \
  [--page-size N] [--max-items N] [--cursor X] [--json]
npx zapier-sdk update-table-records <table> <records-json> [--key-mode names|ids]
npx zapier-sdk delete-table-records <table> <records>
```

Max 100 records per create/update call. `--key-mode` controls whether field references use human-readable names (default) or internal IDs (f1, f2, etc.). Note: `--filters` is now an **array** of `{fieldKey, operator, value}` conditions, and `--sort` is a single `{fieldKey, direction}` object — the older `--filters '{"Status":"Active"}' --sort "field" --direction desc` shape is no longer supported.

---

## Utility Commands

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

---

## Trigger Commands (Experimental, Closed Beta)

> ⚠️ Closed beta. Run via the `zapier-sdk-experimental` bin (or pass `--experimental` to `zapier-sdk`). Flags and behaviour may change. [Request access](https://npsup.zapier.app/contact-us?product=Zapier%20SDK).

### Inbox lifecycle
```bash
npx zapier-sdk create-trigger-inbox <app> <action> [--connection ID] [--inputs '...'] [--notification-url URL]
npx zapier-sdk ensure-trigger-inbox <name> <app> <action> [--connection ID] [--inputs '...'] [--notification-url URL]
npx zapier-sdk get-trigger-inbox <inbox>
npx zapier-sdk list-trigger-inboxes [--name X] [--status X] [--page-size N] [--max-items N] [--cursor X] [--json]
npx zapier-sdk update-trigger-inbox <inbox> [--notification-url URL]
npx zapier-sdk pause-trigger-inbox <inbox>
npx zapier-sdk resume-trigger-inbox <inbox>
npx zapier-sdk delete-trigger-inbox <inbox>
```

`ensure-trigger-inbox` is idempotent on `(user, account, name)` — recommended for production.

### Message handling
```bash
npx zapier-sdk lease-trigger-inbox-messages <inbox> [--lease-limit N] [--lease-seconds N]
npx zapier-sdk ack-trigger-inbox-messages <inbox> <lease> [--messages id1,id2]
npx zapier-sdk release-trigger-inbox-messages <inbox> <lease> [--messages id1,id2]
npx zapier-sdk list-trigger-inbox-messages <inbox> [--page-size N] [--max-items N] [--cursor X]
```

### Drain / watch loops
```bash
npx zapier-sdk drain-trigger-inbox <inbox> \
  [--concurrency N] [--lease-limit N] [--lease-seconds N] \
  [--release-on-error] [--continue-on-error] \
  [--max-messages N] \
  [--exec ./handler | --exec-shell "cmd" | --json]

npx zapier-sdk watch-trigger-inbox <inbox> \
  [--concurrency N] [--lease-limit N] [--lease-seconds N] \
  [--release-on-error] [--continue-on-error] \
  [--max-drain-interval-seconds 60] \
  [--exec ./handler | --exec-shell "cmd" | --json]
```

The three handler modes (`--exec`, `--exec-shell`, `--json`) are mutually exclusive. `--exec` runs a binary per message (no shell). `--exec-shell` runs a shell command. `--json` emits NDJSON to stdout, acking as each write completes.

### Trigger input discovery
```bash
npx zapier-sdk list-trigger-input-fields <app> <action> [--connection ID] [--inputs '...']
npx zapier-sdk get-trigger-input-fields-schema <app> <action> [--connection ID] [--inputs '...']
npx zapier-sdk list-trigger-input-field-choices <app> <action> <input-field> [--connection ID] [--inputs '...']
```

