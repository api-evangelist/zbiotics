---
name: zbiotics-catalog-browse
description: >-
  Find ZBiotics products and read their detail through the store's Universal Commerce Protocol MCP
  endpoint, without spending money or mutating any state. Use this before any cart or checkout
  work, and use it on its own when the user only wants to know what ZBiotics sells or what it costs.
api: ZBiotics Universal Commerce MCP API
endpoint: https://zbiotics.com/api/ucp/mcp
transport: mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
read_only: true
generated: '2026-09-05'
method: generated
source: mcp/zbiotics-mcp-tools.json (live tools/list, fetched 2026-09-05)
---

# Browse the ZBiotics catalog

Every tool name and every field below was read from the live `tools/list` document at
`https://zbiotics.com/api/ucp/mcp`. Nothing here is invented.

## Before you call anything

Every tool requires `meta["ucp-agent"]["profile"]` — a URI pointing at **your** UCP agent profile
document. The store performs a live outbound `GET` of that URI on every `tools/call`. If you omit
it you get `-32001 invalid_profile_url` (HTTP 422); if you supply a URI the store cannot fetch you
get `-32001 profile_unreachable` (HTTP 422). Publish the profile first, then call.

`tools/list` and `initialize` are anonymous and need no profile — use them to confirm the surface
is up.

## Search

```json
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{
  "name":"search_catalog",
  "arguments":{
    "meta":{"ucp-agent":{"profile":"https://your-agent.example/profile.json"}},
    "catalog":{
      "query":"pre-alcohol",
      "context":{"address_country":"US","currency":"USD","language":"en"},
      "filters":{"price":{"min":1000,"max":12000}}
    }
  }}}
```

- `context.address_country` is ISO 3166-1 alpha-2, `currency` is ISO 4217, `language` is BCP 47.
  Pass them — the store uses them for pricing and availability, and says unsupported hints are
  ignored without error rather than rejected.
- `filters.price.min` / `.max` are **integers in minor units**. `1000` is $10.00.

## Look up by id, or fetch one product

- `lookup_catalog` takes `catalog.ids`, an array of **1 to 10** product ids (`minItems: 1`,
  `maxItems: 10`). Batch, do not loop.
- `get_product` takes the same `catalog` object and returns a single complete product.

Ids are opaque and issued by the store. Get them from `search_catalog`. Never construct one, and
never assume the storefront URL handle (`zbiotics`, `sugar-to-fiber`) is the id — they are
different identifier spaces.

## Reading prices back to a user

Prices come back as `{"amount": 600, "currency": "USD"}` — integer minor units. Divide by 100 for
two-decimal currencies before you quote anything. Zero-decimal currencies such as JPY are already
whole units. The store repeats this rule in every tool description because getting it wrong quotes
a user a price 100× off.

## What you cannot do here

There is no paging. `search_catalog` exposes no cursor, limit or offset, so you get what the store
gives you and cannot walk a large result set. If you need the whole catalogue, the provider
documents a read-only HTTP route for it in its own `llms.txt`:
`GET https://zbiotics.com/collections/all/products.json`.

## Errors you will actually see

| code | message | HTTP | what to do |
|---|---|---|---|
| `-32600` | Invalid Request | 400 | Your body is not valid JSON. |
| `-32001` `invalid_profile_url` | UCP discovery failed | 422 | Add `meta["ucp-agent"].profile`. |
| `-32001` `profile_unreachable` | UCP discovery failed | 422 | Your profile URI does not resolve. Serve it. |
