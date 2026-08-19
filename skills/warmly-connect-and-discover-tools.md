---
name: Connect to Warmly and discover its tools
description: Authenticate against Warmly's agent-tools surface over MCP or REST, pin an organization, and read the authoritative tool list with its JSON Schemas before calling anything.
api: https://opps-api.getwarmly.com/api
mcp: https://opps-api.getwarmly.com/api/mcp
operations:
  - GET /agent-tools/tools
  - POST /agent-tools/execute
tools:
  - get_credits_remaining
generated: '2026-08-13'
method: generated
source: https://help.warmly.ai/articles/4646691220-warmly-technical-documentation-mcp-server
---

# Connect to Warmly and discover its tools

Warmly does not expose one endpoint per capability. It exposes a **tool registry** behind a
single dispatcher. Before calling anything, read the registry — it is the only place the real
input contract for each tool is published.

## 1. Pick a surface

**MCP (preferred for agents).** The server is remote and OAuth-protected:

```
claude mcp add --transport http warmly https://opps-api.getwarmly.com/api/mcp
```

The first call returns `401` with a `WWW-Authenticate: Bearer` header carrying
`resource_metadata="https://opps-api.getwarmly.com/.well-known/oauth-protected-resource"`.
Follow that pointer (RFC 9728) to the authorization server, then run authorization-code with
PKCE `S256`. Request `offline_access` if the session must outlive the access token.

**REST.** Send a per-organization API key issued from the Warmly admin UI:

```
Authorization: Bearer $WARMLY_API_KEY
```

## 2. Pin exactly one organization

Every call is scoped to a single organization. Pick one mechanism and use it consistently:

- `organizationId` in the request body (REST)
- `X-Warmly-Organization-Id` header
- `?organization_id=<id>` appended to the MCP URL

If the user belongs to more than one Warmly organization and you do not pin one, you cannot
rely on which workspace's data you get back.

## 3. Read the tool registry

```
GET /agent-tools/tools?organizationId=<id>
```

Returns an array of `{ name, description, isAsync, inputSchema }`. **`inputSchema` is a JSON
Schema and it is authoritative** — prefer it over any documentation, including this skill.
`isAsync` tells you whether step 4 needs polling.

## 4. Verify the connection cheaply

Call `get_credits_remaining`. It is the only documented tool that costs no credits, so it is
the correct health check:

```
POST /agent-tools/execute
{ "toolName": "get_credits_remaining", "organizationId": "<id>", "input": {} }
```

## Rules that apply to every Warmly call

- **HTTP 202 does not mean success.** `POST /agent-tools/execute` returns `202` on every
  outcome. Read `status` (`pending`, `running`, `awaiting_approval`, `succeeded`, `failed`,
  `cancelled`) and `output.success`. On failure, the reason is in `userFacingError`.
- **There is no idempotency key.** `externalRef` is documented only as reference tracking, not
  as a deduplication key. Do not assume a retry is safe, especially for write tools that push
  records into a connected CRM.
- **Rate limits are 60 req/min (free) and 120 req/min (paid), and 429 carries no headers.**
  No `Retry-After` or `RateLimit-*` is published, so back off on your own schedule.
- **Input caps:** `input` is limited to 256 KB serialized, 1,000 keys, 32 levels deep;
  `reasoning` to 50,000 characters.
- **Use `reasoning`.** It stores why the agent made the call on the execution record. It is
  optional but it is what makes the audit trail readable.
