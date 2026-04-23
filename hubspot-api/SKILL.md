---
name: hubspot-api-client
description: Write and execute Python scripts that call the HubSpot API directly using the `hubspot-api-client` SDK. Use whenever the HubSpot MCP connector is too limited — specifically when editing records, creating or editing properties, managing lists/segments, reading or modifying workflows (flows), enrolling contacts in sequences, managing associations and pipelines, logging or reading engagements (calls/emails/notes/meetings), or working with marketing assets (emails, campaigns, forms). Trigger whenever the user asks to bulk-update, export, migrate, audit, or programmatically configure anything in a HubSpot portal. Also trigger when the task would require more than ~5 sequential MCP calls, when the user wants to create or modify a custom property, when the user mentions HubSpot lists/segments/workflows/flows/sequences, or when the user references the HubSpot API or private app token. Do NOT trigger for simple single-record reads — the HubSpot MCP connector handles those fine.
---

# HubSpot API Client — Direct SDK Access

## When to use this skill

Use this skill instead of the HubSpot MCP connector when:

- **Editing records** — MCP is read-heavy; this skill handles `PATCH` and batch updates
- **Properties** — creating, editing, adding enumeration options, archiving properties
- **Lists / segments** — creating lists, updating filter definitions, adding/removing members, pulling memberships
- **Workflows / flows** — reading, creating, updating, or deleting workflows via the v4 Automation API
- **Sequences** — enrolling or unenrolling contacts (subject to Sales/Service seat requirements)
- **Associations & pipelines** — association labels, pipeline stages, default association types
- **Engagements** — logging or reading calls, emails, notes, meetings, tasks
- **Marketing** — single-send/marketing emails, campaigns, forms
- **Bulk operations** — anything above ~50 records; batch endpoints are essential
- **Cross-portal work** — migrating config from one portal to another

For simple one-off reads (e.g. "look up this contact by email"), use the HubSpot MCP connector directly.

## Workflow

1. **Install the SDK** (once per session):
   ```bash
   pip install hubspot-api-client --break-system-packages
   ```

2. **Check authentication** — read the private app token from an env var:
   ```python
   import os
   token = os.environ.get("HUBSPOT_ACCESS_TOKEN")
   if not token:
       raise SystemExit("HUBSPOT_ACCESS_TOKEN environment variable is not set.")
   ```

3. **Confirm the target portal** with the user — ask which client's HubSpot this is for (Terrascope, Knoxx, etc.) and verify the token in scope matches that portal. Private app tokens are prefixed with `pat-<region>-` where region indicates the portal's data centre (e.g. `pat-na1-`, `pat-na2-`, `pat-eu1-`). New regions are added periodically, so don't hardcode prefix checks.

4. **Read the relevant reference file** (see table below) — do not guess endpoints; they change.

5. **Write a purpose-built script** — pagination, rate-limiting, and error handling baked in. Save the script to a scratch location appropriate for the environment (Cowork's session scratch directory, Claude Code's current working directory, claude.ai's `/home/claude/`). Save any deliverables (CSVs, JSON exports, schema dumps) to a location the user can access and open — the exact path convention depends on the environment.

6. **Execute and present results** — always show a summary of what was read/written. For writes, show a dry-run first unless the user has already confirmed. Surface deliverable files using whatever mechanism the environment provides (`computer://` links in Cowork, `present_files` in claude.ai, plain paths in Claude Code).

## Authentication

- Always read the token from `os.environ["HUBSPOT_ACCESS_TOKEN"]`; never hardcode
- If querying a client portal, confirm with the user which portal the token belongs to before running anything that writes
- Private app tokens are scoped — if a call fails with `MISSING_SCOPES`, report the required scope to the user so they can add it in the HubSpot private app settings
- **Never use hapikey** (API key) — deprecated and unsupported after SDK v5.1.0
- OAuth flows are out of scope for this skill; use the `OAuthClient` class directly if needed

## Client initialisation (standard pattern)

```python
import os
from hubspot import HubSpot
from urllib3.util.retry import Retry

# Retry on 429 (rate limit) and 5xx with exponential backoff
retry = Retry(
    total=5,
    backoff_factor=0.3,
    status_forcelist=(429, 500, 502, 503, 504),
)

client = HubSpot(access_token=os.environ["HUBSPOT_ACCESS_TOKEN"], retry=retry)
```

Use this as the top of every script. The SDK has built-in pagination helpers for `search_api.do_search()`, but most other endpoints require manual pagination (see `references/core-patterns.md`).

## Reference files

Read the file that matches the task before writing code. Each file has copy-paste-ready patterns tested against the 2026-03 API version.

| File | When to read |
|---|---|
| `references/core-patterns.md` | **Always** — pagination, rate limiting, batch operations, error handling, the `api_request` escape hatch for endpoints the SDK doesn't wrap |
| `references/crm-objects.md` | Reading, creating, editing, searching, deleting contacts / companies / deals / tickets / custom objects; batch upserts |
| `references/properties.md` | Creating and editing properties, adding dropdown/radio options, property field types, archiving, property groups |
| `references/lists.md` | Creating static and dynamic lists, updating filter definitions (the `filterBranch` syntax), adding/removing members, fetching memberships |
| `references/workflows.md` | The v4 Automation (flows) API — reading, creating, updating flows; enrollment settings; mapping from v3 IDs |
| `references/sequences.md` | Enrolling and unenrolling contacts; seat and inbox prerequisites; common 500-error causes |
| `references/associations.md` | Association labels (v4), default vs custom association types, setting associations in bulk, pipelines and stages |
| `references/engagements.md` | Logging calls, emails, notes, meetings, tasks; reading engagement history; associating engagements to records |
| `references/marketing.md` | Marketing emails (single-send and transactional), campaigns, forms, subscription status |

**For any task, always read `core-patterns.md` first**, then the area-specific file(s).

## API versioning (2026-03)

HubSpot introduced date-based API versioning in March 2026. New endpoints follow the pattern `/{api}/2026-03/{resource}`. The Python SDK still exposes endpoints under the legacy v3/v4 paths internally, which continue to work and are supported through March 2027. For new direct HTTP calls via `client.api_request()`, prefer `/2026-03/` paths. The reference files note which pattern to use for each area.

## Rate limits (know before you bulk)

- **Private apps, Pro subscription**: 190 requests per 10 seconds, 650k/day
- **Private apps, Enterprise**: 190 requests per 10 seconds, 1M/day
- **Search API**: 5 requests/second, capped at 200 records per response, **max 10,000 records per query** (paginate by `hs_object_id` to exceed)
- **Sequences enrollment**: effectively per-user; requires a connected inbox on the acting user
- Daily quota is shared across all private apps in the portal

Always check `X-HubSpot-RateLimit-Remaining` headers when running bulk jobs. Use `references/core-patterns.md` rate-limiter helpers.

## Key constraints

- **Always paginate** — `basic_api.get_page()` returns max 100; search returns max 200
- **Search API has a hard 10,000-record cap per query** — break large pulls by `hs_object_id` ranges or `createdate` windows
- **Search index is eventually consistent** — freshly written records may not appear for a few seconds. For time-sensitive reads after writes, use `basic_api.get_by_id()` or `batch_api.read()` instead of search
- **Batch endpoints cap at 100 records** per call for most objects
- **Writes are not dry-runnable** — the API has no "preview" mode. For destructive or bulk writes, always produce a dry-run summary (e.g. "would update 347 contacts; first 5: …") and get explicit confirmation before executing
- **Workflow PUTs are destructive** — omitted fields are removed. Always fetch the current state first, mutate, and PUT the full object back
- **Associations have two kinds of types**: default `HUBSPOT_DEFINED` types and `USER_DEFINED` labels. Mixing them up will silently fail — see `references/associations.md`

## Common tasks

| User request | Reference file | Approach |
|---|---|---|
| "Bulk update a property on 500 contacts" | crm-objects | Batch update in chunks of 100 |
| "Create a custom property called X" | properties | Single POST with correct `type` + `fieldType` pair |
| "Add options to this dropdown property" | properties | PATCH the property with the full updated `options` array |
| "Export all deals in the pipeline" | crm-objects | Paginated `basic_api.get_page()` or `search_api.do_search()` |
| "Create a list of contacts matching X" | lists | POST to `/crm/v3/lists/` with a `filterBranch` definition |
| "Enrol these contacts in sequence Y" | sequences | Per-contact POST; handle seat/inbox errors |
| "Clone this workflow to another portal" | workflows | GET flow → strip IDs/timestamps → POST as new flow |
| "Log a call on this contact" | engagements | Create call engagement + associate to contact |
| "Send a single-send marketing email" | marketing | Single-send API with contact email ID |
| "Find contacts in list X but not list Y" | lists + crm-objects | Fetch memberships → set difference → batch read |

## Output and presentation

- For read tasks: display a summary (count, sample) in chat; if large, save the full output as CSV to a location the user can access and surface it with the environment's file-sharing mechanism
- For write tasks: report what was changed (count, IDs, field values), and flag any records that failed with the HubSpot error message
- For schema/config exports (properties, workflow JSON, list definitions): save to markdown or JSON so Dennis can diff across portals; put them somewhere openable (not a scratch-only location that disappears at session end)

## Relationship to other skills

- **HubSpot MCP connector** — use for simple reads, lookups by ID, quick record checks. This skill takes over for bulk, writes, schema changes, and areas MCP doesn't cover (workflows, lists, sequences, property management)
- **`zapier-platform-cli`** / **`zapier-platform-ui`** — use when the task is better served by a Zapier integration (recurring automation, no-code team handoff). Use this skill for one-off scripts and audits
- **`linear-client-routing`** — if work for Terrascope produces tickets, route them to the correct Linear team
