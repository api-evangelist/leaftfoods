---
name: leaftfoods-agent-purchase
description: Buy Leaft Foods products on behalf of a consenting human buyer using the store's Universal Commerce Protocol MCP endpoint — discover capabilities, search the catalog, build a cart, open a checkout, set fulfillment, and complete payment only after explicit buyer approval.
api: leaftfoods:ucp-commerce
generated: '2026-07-19'
method: generated
source: https://www.leaftfoods.com/llms.txt
operations:
  - search_catalog
  - create_cart
  - create_checkout
  - update_checkout
  - complete_checkout
---

# Purchase from Leaft Foods as an agent

Leaft Foods runs a Shopify-native [Universal Commerce Protocol](https://ucp.dev) store.
All transacting happens over one JSON-RPC 2.0 endpoint.

- Discovery: `GET https://www.leaftfoods.com/.well-known/ucp`
- MCP endpoint: `POST https://www.leaftfoods.com/api/ucp/mcp` with `Content-Type: application/json`

The five tool names below are the ones the store documents in its own `llms.txt`. Confirm
their exact schemas with the MCP `tools/list` method before calling them — do not assume
argument shapes.

## Before you start

You must be able to obtain **contemporaneous buyer approval at the moment of payment**.
If you cannot, stop here and route the purchase through the Shop skill
(`https://shop.app/SKILL.md`) and Shop Pay instead — that is what the store asks you to do.

You must also present a resolvable **agent profile URI** during UCP discovery. Calling the
endpoint without one returns HTTP 422 with UCP error `invalid_profile_url`
(`-32001`, "Missing profile uri").

## Steps

1. **Discover.** `GET /.well-known/ucp`. Read `ucp.version` and pick a version from
   `supported_versions` (currently `2026-04-08` latest stable, `2026-01-23` also supported).
   Confirm the capabilities you need are present: `dev.ucp.shopping.catalog.search`,
   `dev.ucp.shopping.cart`, `dev.ucp.shopping.checkout`, `dev.ucp.shopping.fulfillment`.

2. **List tools.** Call MCP `tools/list` to get the authoritative tool schemas for the
   version you selected.

3. **Search.** Call `search_catalog` with the buyer's intent. Always pass buyer context —
   `context.address_country` and `context.currency` — or pricing and availability will be
   wrong.

4. **Cart.** Call `create_cart` with the chosen items and quantities.

5. **Checkout.** Call `create_checkout` to open the purchase flow against the cart.

6. **Fulfill.** Call `update_checkout` to set the shipping address and shipping method.
   Note the store's fulfillment config: `allows_multi_destination.shipping` is `false`, and
   the only allowed method combination is `["shipping"]`. One destination, shipping only.

7. **Approve, then complete.** Present the final total, line items, shipping and any
   discount to the human. Only after they explicitly approve, call `complete_checkout`.
   Payment handlers available are Google Pay (`gpay`, gateway `shopify`; VISA, MASTERCARD,
   AMEX, DISCOVER) and Shopify card (`shopify.card`; visa, master, american_express,
   discover, diners_club).

## Error handling

Errors come back as JSON-RPC 2.0 error objects, **not** RFC 9457 problem+json:

```json
{"jsonrpc":"2.0","id":1,"error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"...","continue_url":"https://..."}}}
```

- Read `error.data.code` for the machine-readable slug.
- When `error.data.continue_url` is present, that is a URL a human can open to finish the
  flow — surface it to the buyer rather than retrying blindly.
- On **429**, back off. The endpoint is rate-limited per IP and no `Retry-After` contract
  is published, so use exponential backoff.
- There is **no idempotency-key contract** on this surface. Do not blind-retry
  `create_cart`, `create_checkout` or `complete_checkout` — re-discover state first.

See `errors/leaftfoods-problem-types.yml` and `conventions/leaftfoods-conventions.yml`.
