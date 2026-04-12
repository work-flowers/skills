# work.flowers Workspace Schema

This file contains database IDs and relation maps for Dennis's own Notion workspace (work.flowers CRM). **Only read this file when querying Dennis's workspace databases.** For client workspaces or unknown databases, use `notion.databases.retrieve()` to discover the schema dynamically — see SKILL.md for the pattern.

## Database IDs

Use these with `notion.databases.query(database_id=...)`.

| Database | database_id |
|---|---|
| **Companies** | `21991b07-11ac-80b0-b787-000b3d3995f6` |
| **Contacts** | `21991b07-11ac-81a6-a894-000be4a09a67` |
| **Deals** | `21a91b07-11ac-808d-9657-000b1390d20b` |
| **Meeting Notes** | `19891b07-11ac-8137-9d62-000b75fab86e` |
| **Emails** | `1e491b07-11ac-80ce-8b86-000b29ba4f68` |
| **Leads** | `21191b07-11ac-81df-bb14-000b500ff9b5` |
| **Sales Invoices** | `21a91b07-11ac-80b0-bbf7-000b242301c1` |
| **Project Proposals** | `1d791b07-11ac-8058-8c70-000b0d0dfaf2` |
| **Tasks** | `27a91b07-11ac-81ed-973f-000ba6da1441` |
| **Google Drive Files** | `1e691b07-11ac-801b-9f84-000bfe4e4143` |
| **Related Docs** | `12191b07-11ac-8075-b4dc-000b66df7481` |

## Python constants

```python
DB = {
    "companies":         "21991b07-11ac-80b0-b787-000b3d3995f6",
    "contacts":          "21991b07-11ac-81a6-a894-000be4a09a67",
    "deals":             "21a91b07-11ac-808d-9657-000b1390d20b",
    "meeting_notes":     "19891b07-11ac-8137-9d62-000b75fab86e",
    "emails":            "1e491b07-11ac-80ce-8b86-000b29ba4f68",
    "leads":             "21191b07-11ac-81df-bb14-000b500ff9b5",
    "sales_invoices":    "21a91b07-11ac-80b0-bbf7-000b242301c1",
    "project_proposals": "1d791b07-11ac-8058-8c70-000b0d0dfaf2",
    "tasks":             "27a91b07-11ac-81ed-973f-000ba6da1441",
    "google_drive":      "1e691b07-11ac-801b-9f84-000bfe4e4143",
    "related_docs":      "12191b07-11ac-8075-b4dc-000b66df7481",
}
```

---

## Relation map

### Companies (primary hub — richest relations)

| Relation Property | Points To | Database Key |
|---|---|---|
| `Deals` | Deals | `deals` |
| `Meeting Notes` | Meeting Notes | `meeting_notes` |
| `Emails` | Emails | `emails` |
| `Clay Contacts` | Contacts | `contacts` |
| `Project Proposals` | Project Proposals | `project_proposals` |
| `Sales Invoices` | Sales Invoices | `sales_invoices` |
| `Legal Agreements` | Legal Agreements | — |
| `Library` | Library docs | — |
| `Readwise Highlights` | Readwise | — |
| `Primary Billing Contact` | Contacts | `contacts` |
| `💼 Docs` | Documents | — |
| `🧾 Project Addendums` | Project Addenda | — |

### Contacts

| Relation Property | Points To |
|---|---|
| `Related Company` | Companies |
| `Deals` | Deals |
| `Deals Referred` | Deals |
| `Meeting Notes` | Meeting Notes |
| `📥 Emails` | Emails |
| `⚖️ Signed Legal Agreements` | Legal Agreements |
| `Customer Reviews` | Reviews |
| `Duplicate of` / `Duplicated by` | Other Contacts |

### Deals

| Relation Property | Points To |
|---|---|
| `Company` | Companies |
| `Contact` | Contacts |
| `Meeting Notes` | Meeting Notes |
| `Emails` | Emails |
| `Project Proposal` | Project Proposals |
| `SOWs` | SOWs / Project Addenda |
| `Tasks` | Tasks |
| `⚖️ Signed Legal Agreements` | Legal Agreements |
| `Referred by` | Contacts |

### Meeting Notes

| Relation Property | Points To |
|---|---|
| `Companies` | Companies |
| `Contacts` | Contacts |
| `Deals` | Deals |
| `Tasks` | Tasks |
| `Google Drive Files` | Google Drive Files |
| `Related Docs` | Related Docs |

### Emails

| Relation Property | Points To |
|---|---|
| `Companies` | Companies |
| `Contacts` | Contacts |
| `Deals` | Deals |

---

## Lookup strategy

| User asks for... | Start from | Follow relation |
|---|---|---|
| Meeting notes for a company | Companies → find by name | `Meeting Notes` |
| Emails with a company | Companies → find by name | `Emails` |
| Deals for a company | Companies → find by name | `Deals` |
| Proposals sent to a company | Companies → find by name | `Project Proposals` |
| Invoices for a company | Companies → find by name | `Sales Invoices` |
| Meeting notes for a contact | Contacts → find by name | `Meeting Notes` |
| Emails from/to a contact | Contacts → find by name | `📥 Emails` |
| All context for a deal | Deals → find by name | `Meeting Notes`, `Emails`, `Project Proposal`, `SOWs`, `Tasks` |
| Tasks linked to a meeting | Meeting Notes → find by title | `Tasks` |

---

## Important notes

- **Companies is the best anchor** for most lookups — it has the most relations
- **Deals connect to delivery context** — through Tasks, SOWs, and Project Proposals
- **Relations are bidirectional** — if Companies has `Meeting Notes`, then Meeting Notes has `Companies`
- **Relation properties** appear as arrays of `{"id": "page-id"}` objects in SDK query results
- **Rollup properties** (e.g. `Company Name` on Deals) are derived from relations — they confirm the link but the actual linked records come from the relation property
- To find a company by name, query the Companies database with a title filter matching the company name
