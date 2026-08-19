---
name: Pull identified website visitors and accounts
description: Retrieve de-anonymized website visitors and their company-level rollups from Warmly, filtered to an ICP, while controlling credit spend and handling the always-202 execution record correctly.
api: https://opps-api.getwarmly.com/api
mcp: https://opps-api.getwarmly.com/api/mcp
operations:
  - POST /agent-tools/execute
  - GET /agent-tools/executions/:id
tools:
  - list_warm_visitors
  - list_warm_accounts
generated: '2026-08-13'
method: generated
source: https://help.warmly.ai/articles/4646691220-warmly-technical-documentation-mcp-server
---

# Pull identified website visitors and accounts

This is Warmly's core capability: turning anonymous web traffic into named people and named
companies. Two tools cover it — `list_warm_visitors` returns individuals,
`list_warm_accounts` returns the same population grouped by company.

**This costs money.** Warmly charges 1 credit per newly identified company and 1 per newly
identified contact, capped at 2 credits per unique visitor per calendar month. Re-reading
someone you have already identified this month is free. Filter before you call, not after.

## Prerequisites

Complete `warmly-connect-and-discover-tools` first. Read the live `inputSchema` for both
tools from `GET /agent-tools/tools` — the parameters below are what Warmly documents, but
the schema is authoritative.

## 1. Narrow to the ICP first

`list_warm_visitors` accepts these documented filters. Use them; a broad call pays for rows
you will discard.

| Parameter | Purpose |
|---|---|
| `timeWindow` | How far back to look |
| `take` | Page size, 1–500 |
| `offset` | Pagination offset |
| `requireBusinessContact` | Drop rows with no business contact |
| `searchTerm` | Free-text match |
| `countries` | Geography filter |
| `industries` | Industry filter |
| `employeeSizeBands` | Company-size filter |
| `pagesVisitedContains` | Intent filter — the highest-signal one |

`pagesVisitedContains` is usually the sharpest filter available: someone who reached the
pricing page is a different prospect from someone who read a blog post.

## 2. Execute

```
POST /agent-tools/execute
{
  "toolName": "list_warm_visitors",
  "organizationId": "<id>",
  "reasoning": "Weekly ICP pull for enterprise pricing-page visitors",
  "input": {
    "timeWindow": "<window>",
    "take": 100,
    "requireBusinessContact": true,
    "pagesVisitedContains": ["/pricing"]
  }
}
```

The response is **always HTTP 202**. Check `status` and `output.success` — never the HTTP
status — and read `userFacingError` when `status` is `failed`.

## 3. Poll only if the registry said the tool is async

`GET /agent-tools/tools` reports `isAsync` per tool. The read tools are documented as
synchronous and return inline. If `isAsync` is true, poll:

```
GET /agent-tools/executions/:id
```

until `status` is one of `succeeded`, `failed` or `cancelled`. Treat `awaiting_approval` as
non-terminal — a human has to act before it moves.

## 4. Page

Increment `offset` by `take`. Warmly publishes no total count and no next-page cursor, so
**stop when a page returns fewer rows than `take`.** That is the only end-of-list signal.

## 5. Roll up to accounts when you want companies, not people

`list_warm_accounts` takes the same filter set and returns companies with aggregated session
counts and distinct-page metrics. Prefer it for account-based routing and for prioritising
outreach — it is far cheaper per unit of decision than paging individuals.

## What you get back

- **Session:** `lastSeen`, `totalSessions`, `sessionTime`
- **Company:** `name`, `domain`, `industry`, `employeeRange`, `revenue`
- **Contact:** `name`, `title`, `seniority`, `linkedInHandle`, `workEmail`
- **CRM intersection:** whether the record already exists in HubSpot, Salesforce or Pipedrive

Check the CRM intersection before you write anything back. Warmly is a join between its own
identity graph and the customer's CRM, and creating a duplicate is the common failure.

## Cautions

- `domain` is the only reliable key on an account; **no stable public identifier is documented
  for an individual visitor**, so you cannot re-fetch one by id — only re-run a filtered query.
- Handle nulls. Enriched fields come back `null` or empty when unavailable.
- This data is about identified people. Respect the customer's own consent and privacy posture
  before pushing it anywhere.
