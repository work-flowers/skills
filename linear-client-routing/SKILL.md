---
name: linear-client-routing
description: Reference for routing Linear issues to the correct team based on client. Use this skill whenever creating or updating Linear issues for a work.flowers client — it provides the team ID, key, and name for each client so issues land in the right place. Trigger when the user mentions creating a Linear issue, task, or ticket for any client listed here.
---

# Linear Client Routing

When creating Linear issues for work.flowers clients, always look up the correct team from the table below. Use the `id` value as the `team` parameter when calling Linear tools.

## Client → Team Mapping

| Key    | Client Name                        | Team ID                              |
| ------ | ---------------------------------- | ------------------------------------ |
| COGROW | co:grow                            | d74c31be-037d-4b02-b5fb-25e2e4dadf6b |
| EAA    | Elite Athlete Academy              | 13437119-f2c0-4b3a-b9da-6ce31cd984b7 |
| EFAI   | EliteFit AI                        | 30ae0117-d937-4cb8-9e90-f163e45edafa |
| GGV    | Golden Gate Ventures               | 36053475-0bc5-4baa-8372-3c3c5aad3d1b |
| KNOXX  | Knoxx Business Group Pty Ltd       | d401de3e-0e04-49bd-8bed-02c3227e827c |
| MHA    | Ministry of Home Affairs Singapore | cdd76c98-e4b6-4a86-bc60-effc93000449 |
| NTUC   | NTUC                               | d19db0ae-0f6a-40c2-96a1-970999087d3f |
| SCW    | Secure Code Warrior                | 5706e4ec-8899-47be-9648-9d03edc58cb2 |
| SKNA   | Sakana AI                          | d13c6be4-526d-4a10-9157-a9dd91f2a83f |
| STA    | Start Right                        | 35200f37-11fc-46fb-ae66-7d69d9b79a7b |
| TS     | Terrascope                         | 88464f6d-d5a9-4d2b-8b93-037d7e50a566 |
| WFOF   | Ordinary Folk                      | 8378580a-ac49-449d-b168-c931135f103f |

## Usage Rules

1. **Match by any identifier** — the user may refer to a client by full name, key, or abbreviation (e.g. "EF", "EliteFit", "EFAI" all map to EliteFit AI).
2. **Always use the Team ID** — pass the UUID from the table as the `team` parameter in `Linear:create_issue` or `Linear:update_issue`.
3. **Default assignee** — unless the user specifies otherwise, assign all new issues to **Dennis Chiuten** (user ID: `bc780c50-f861-47cd-b7b6-e5dd734a75ce`). Pass this UUID as the `assignee` parameter when calling Linear tools.
4. **Ask if ambiguous** — if the user's reference could match multiple clients, ask which one they mean before creating the issue.
5. **Unknown clients** — if a client isn't in this table, ask the user which Linear team to use rather than guessing.

## Special Routing: Knoxx Foods

For **Knoxx Foods** (KNOXX) issues, **do not use the standard Linear MCP server**. Instead, use the **Zapier MCP connector** which provides tools for managing issues in Knoxx's Linear workspace. This gives you access to Knoxx-specific integrations and workflow automations that aren't available through the standard Linear tools.

When working on Knoxx issues:
- Use `Zapier:linear_create_issue`, `Zapier:linear_update_issue`, `Zapier:linear_find_issue_by_id`, or `Zapier:linear_find_issues_by_name` instead of the standard `Linear:*` tools
- You do not need to provide a team ID when using Zapier tools — they route directly to Knoxx's workspace
- Reference the Team ID above (KNOXX: d401de3e-0e04-49bd-8bed-02c3227e827c) only if explicitly needed for context or documentation purposes