---
name: tellius-ask-a-governed-question
description: Ask Tellius a natural-language question about governed enterprise data through MCP and read the answer safely, validating the generated SQL before displaying any number.
api: Tellius MCP Server
generated: '2026-08-30'
method: generated
source: https://help.tellius.com/kaiya/tellius-mcp-server
operations:
  - tellius_list_business_views
  - tellius_get_business_view_details
  - tellius_ask
  - tellius_get_starter_questions
---

# Ask Tellius a governed question

Tellius answers questions against a **Business View** — a governed semantic model that
carries the metric definitions, joins, hierarchies and row-level security. The answer
you get is the same answer the same question gets inside the product, scoped to the
permissions of the user whose credentials the connection holds.

## Before you start

- The MCP endpoint is your deployment's own host, ending in `/mcp`. Ask the Tellius
  administrator for it; there is no shared public URL.
- Authenticate with browser sign-in (OAuth authorization code, PKCE S256) where the
  client supports it, or an administrator-issued client id and secret where it does not.
  The only scope is `mcp`.

## Steps

1. **Find the Business View.** Call `tellius_list_business_views` and page through the
   results. Take the `business_view_id`, never the name — names can be edited, ids are
   stable.
2. **Check it holds what you need.** Call `tellius_get_business_view_details` for the
   full column schema, data types and sample values. If the client supports MCP
   resources, `tellius://business-views/{id}/schema` returns the same thing without a
   tool call.
3. **If you do not know what to ask,** call `tellius_get_starter_questions` for
   suggested questions on that Business View.
4. **Ask.** Call `tellius_ask` with:
   - `question` — plain English, with every filter written into the sentence.
   - `business_view_ids` — a list containing the one id.
   - `include_raw: true` — always, even when you only need a single number.
5. **Write filters into the sentence, not into fields.** There are no structured filter
   parameters. Assemble the question as
   `<metric> [for <product>] [in the <region> region] [for <period>]`. Omit any filter
   set to "All" — asking without naming a product already covers all of them.
6. **Keep wording identical across refreshes.** Use the same phrasing every time for the
   same metric, changing only the filter values. Consistent wording produces consistent
   SQL, which is what keeps a displayed number stable between refreshes.

## Validate before you display

`include_raw: true` returns three parts: a written `summary`, a `data` table with
`columns` and `rows`, and the `sql` Tellius generated and ran.

1. **Confirm you received data, not a follow-up.** The response may come back asking a
   clarifying question, or asking the user to sign in. Treat either as "no value yet"
   and do not render it as a number.
2. **Check the SQL against your intent.** The returned SQL shows the filters and the
   aggregation actually used. This is the strongest guard against a misread question,
   and it is the reason to send `include_raw: true` even for one number.
3. **Sanity-check the value.** Confirm it is numeric and within a plausible range. Where
   the data has a natural hierarchy, do a rollup check — the parts should sum to the
   whole for the same metric and period.

Read the exact field names from the `tellius_ask` output schema shown in your MCP
client; a chart definition may also be returned.

## Error handling

- **401 with `{"error":"invalid_token"}`** — the `WWW-Authenticate` header names a
  `resource_metadata` URL. Fetch `/.well-known/oauth-protected-resource`, then the
  authorization server it points at, and complete the flow requesting scope `mcp`.
- There is no rate-limit header and no documented 429 contract. Back off on your own
  schedule.
- There is no idempotency key on any Tellius write surface, so never blind-retry a
  tool that creates something.

## What the user should know

Every question asked this way is saved as a conversation turn in Tellius and stays
visible in the Kaiya interface, so activity outside the product is auditable alongside
in-product usage.
