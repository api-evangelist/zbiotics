---
name: zbiotics-agent-purchase
description: >-
  Buy ZBiotics products on a user's behalf through the store's Universal Commerce Protocol MCP
  endpoint — cart, checkout, and completion. This skill spends real money on a live store. Use it
  only with contemporaneous buyer approval at the moment of payment, which the provider requires.
api: ZBiotics Universal Commerce MCP API
endpoint: https://zbiotics.com/api/ucp/mcp
transport: mcp
operations:
  - search_catalog
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
read_only: false
irreversible_operations:
  - complete_checkout
generated: '2026-09-05'
method: generated
source: >-
  mcp/zbiotics-mcp-tools.json (live tools/list, fetched 2026-09-05),
  https://zbiotics.com/llms.txt
---

# Purchase from ZBiotics as an agent

Tool names, field names and error codes below were read from the live `tools/list` document and
from ZBiotics' own `llms.txt`. There is no test mode and no sandbox store: **every call in this
skill mutates the live store.**

## Non-negotiables

1. **Human approval at payment.** ZBiotics states it plainly: *"Checkout requires human approval.
   Agents must not complete payment without explicit buyer consent."* If you cannot get
   contemporaneous approval at the moment of payment, the provider's own instruction is to stop
   and route the purchase through Shop Pay via `https://shop.app/SKILL.md` instead.
2. **There is no idempotency.** No `Idempotency-Key`, no client-supplied request id, nothing in
   any of the seven write tools' input schemas. The JSON-RPC `id` field is a response correlator,
   **not** a deduplication key. If `complete_checkout` times out you do **not** know whether the
   order was placed. Do not blind-retry it — read state back with `get_checkout` first.
3. **`complete_checkout` cannot be undone.** `cancel_cart` and `cancel_checkout` exist;
   there is no `cancel_order`, no refund tool and no void tool. Once an order exists the only path
   back is the human refund policy at `https://zbiotics.com/pages/refund-policy`.
4. **Every call needs your agent profile.** `meta["ucp-agent"]["profile"]` must be a URI the store
   can fetch. It dereferences it on every `tools/call`.

## The flow

### 1. Find the variant
Use `search_catalog` (see the `zbiotics-catalog-browse` skill). You need a **product variant id** —
pack sizes are separate variants, and picking the wrong one buys the wrong quantity.

### 2. Create the cart

```json
{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
  "name":"create_cart",
  "arguments":{
    "meta":{"ucp-agent":{"profile":"https://your-agent.example/profile.json"}},
    "cart":{
      "line_items":[{"item":{"id":"<product-variant-id>"},"quantity":1}],
      "buyer":{"email":"buyer@example.com"},
      "context":{"address_country":"US","currency":"USD"}
    }
  }}}
```

`line_items[].item.id` and `line_items[].quantity` are both required. Adjust with `update_cart`
(pass the line's own `id` to change an existing line); abandon with `cancel_cart`.

### 3. Create and fill the checkout

`create_checkout` takes a `checkout` object. Use `update_checkout` to set the shipping address and
the fulfillment method, and to attach a payment instrument. Read the current totals back with
`get_checkout` — it returns line items, totals, discounts and taxes, all as minor-unit integers.

Payment instruments are conditionally shaped. For `handler_id: "apple-pay"` the schema requires
`billing_address` and a `credential` of type `apple_pay_token` carrying `payment_data`. For every
other handler the `credential` requires `token` and `type`. The store declares three handlers in
its `/.well-known/ucp.json`: `com.google.pay`, `dev.shopify.card` and `dev.shopify.shop_pay`.

### 4. Confirm with the human, then complete

Quote the total **in major units** (divide the minor-unit integer by 100 for USD), name the
product and quantity, and get an explicit yes. Then:

```json
{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{
  "name":"complete_checkout",
  "arguments":{"meta":{"ucp-agent":{"profile":"https://your-agent.example/profile.json"}},
               "id":"gid://shopify/Checkout/<id>","checkout":{}}}}
```

It returns an order ID and a Thank You Page URL, or errors. Hand the user the URL.

### 5. Order follow-up needs a customer token

`get_order` takes `gid://shopify/Order/{id}` and refuses anonymous calls with
`-32000 AuthenticationRequired` (HTTP 403). It needs a customer-account JWT, obtained through the
OIDC flow at `https://account.zbiotics.com` with scope `customer-account-mcp-api:full`
(discovery: `https://zbiotics.com/.well-known/openid-configuration`, authorization_code + PKCE
`S256`). Without that token, tell the user to check the Thank You Page URL instead of guessing.

## Rate limits

ZBiotics states the endpoint is rate-limited per IP and to back off on `429`. No number, no
window, and **no rate-limit response headers at all** — so you get no warning before you are
refused. Pace yourself conservatively and treat any `429` as a hard stop, not a retry cue.

## Bailing out safely

| situation | do this |
|---|---|
| User hesitates before payment | `cancel_checkout`, then `cancel_cart`. Both are real tools. |
| `complete_checkout` timed out | `get_checkout` to read actual state. Never blind-retry. |
| Order placed in error | Stop. There is no API reversal. Direct the user to the refund policy. |
