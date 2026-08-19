---
name: Look up third-party intent signals without burning credits
description: Query Warmly's third-party intent signals in both discovery and enrichment modes, using preview mode as a dry run to size a result set before paying for it.
api: https://opps-api.getwarmly.com/api
mcp: https://opps-api.getwarmly.com/api/mcp
operations:
  - POST /agent-tools/execute
  - GET /agent-tools/executions/:id
tools:
  - list_third_party_signals
  - get_credits_remaining
generated: '2026-08-13'
method: generated
source: https://www.warmly.ai/launches/warmly-mcp-and-api-are-live
---

# Look up third-party intent signals without burning credits

`list_third_party_signals` reaches outside the customer's own website into third-party intent:
financial events, hiring trends, leadership changes, news, G2 reviews, SEC filings and
LinkedIn engagement.

It has a property none of Warmly's other tools have: **a dry run.** Use it.

## Two modes, one tool

The mode is an input field, not a separate tool or path.

- **`by_signal`** — discovery. "Which companies are showing this signal?" Start here when you
  are building a target list.
- **`by_company`** — enrichment. "What signals is this company showing?" Start here when you
  already have an account and want to know whether now is the moment.

## Documented filters

| Parameter | Purpose |
|---|---|
| `signalCategory` | Broad grouping |
| `signalType` | Specific signal |
| `signalSubtype` | Narrower still |
| `companyDomain` | Required in practice for `by_company` |
| `detectedSince` | Only signals detected after this point |
| `preview` | Dry run — returns totals, charges nothing |

Read the live `inputSchema` from `GET /agent-tools/tools` before relying on any of these.

## 1. Always preview first

```
POST /agent-tools/execute
{
  "toolName": "list_third_party_signals",
  "organizationId": "<id>",
  "reasoning": "Sizing hiring-signal cohort before committing credits",
  "input": {
    "mode": "by_signal",
    "signalCategory": "<category>",
    "detectedSince": "<timestamp>",
    "preview": true
  }
}
```

Preview returns totals **without charging credits**. If the count is larger than your budget,
tighten `signalType`/`signalSubtype`/`detectedSince` and preview again. Iterate here, where it
is free, rather than on the paid call.

## 2. Check the budget

```
POST /agent-tools/execute
{ "toolName": "get_credits_remaining", "organizationId": "<id>", "input": {} }
```

Free. Compare the remaining balance against the previewed total before step 3. There is no
rate-limit or quota header on any Warmly response, so this tool is the **only** way to read
remaining budget.

## 3. Run for real

Re-send the same input with `preview` removed or set to `false`. Remember the response is
**HTTP 202 regardless of outcome** — read `status` and `output.success`, and
`userFacingError` on failure. Poll `GET /agent-tools/executions/:id` if the registry reported
`isAsync: true`.

## 4. Join back to your accounts

Signals are keyed on `companyDomain`. That is the same key `list_warm_accounts` returns, so
the natural play is: pull accounts that visited the site, then enrich the interesting ones with
`by_company`. First-party visit plus third-party signal is a far stronger trigger than either
alone.

## Cautions

- **No idempotency key exists.** If a call times out, a blind retry may re-run and re-charge.
  Prefer `preview` to establish state before retrying a paid call.
- Rate limits are 60 req/min free, 120 req/min paid, and a `429` arrives with no `Retry-After`.
  Back off on your own schedule.
- Credits reset monthly and the per-visitor dedupe window clears on the calendar month change,
  so identical queries can cost differently either side of month end.
