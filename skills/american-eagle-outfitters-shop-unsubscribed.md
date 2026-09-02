---
name: shop-unsubscribed
description: >-
  Search, cart and check out American Eagle Outfitters' Unsubscribed brand over its live
  UCP/MCP endpoint, respecting the store's published human-approval invariant on payment.
api: Unsubscribed Commerce (UCP/MCP)
endpoint: https://www.unsubscribed.com/api/ucp/mcp
transport: mcp
protocol: Universal Commerce Protocol 2026-08-25 over MCP 2024-11-05
generated: '2026-09-02'
method: generated
source: >-
  mcp/american-eagle-outfitters-unsubscribed-mcp-tools.json (tools/list fetched live
  2026-09-02) and https://www.unsubscribed.com/agents.md
operations:
  - search_catalog
  - lookup_catalog
  - get_product
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
---

# Shopping Unsubscribed as an agent

Unsubscribed is one of American Eagle Outfitters' five brands. Its storefront exposes a
Universal Commerce Protocol shopping server over MCP. Every tool name below was read from a
live `tools/list` on 2026-09-02 — none is invented.

## Before you start

- **Endpoint**: `POST https://www.unsubscribed.com/api/ucp/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`.
- **Discovery is free.** `initialize` and `tools/list` need no credential. Read
  `tools/list` first and use the returned `inputSchema` for each tool — it is the contract.
- **Everything else needs an agent identity.** Every non-discovery call requires
  `params.meta["ucp-agent"].profile` — a resolvable UCP agent profile URI. Without it you
  get HTTP 422 and JSON-RPC `-32001` / `invalid_profile_url`. There is no API key.
- **Confirm the version.** `GET https://www.unsubscribed.com/.well-known/ucp.json` and read
  `ucp.supported_versions`. Three are live: `2026-08-25` (current), `2026-04-08`, `2026-01-23`.

## The one rule that overrides everything else

The store publishes this in both `robots.txt` and `agents.md`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent flows
> that finalize payment without an explicit, contemporaneous human approval step.

So: **never call `complete_checkout` without contemporaneous buyer approval.** Build the
cart, build the checkout, show the buyer the totals, and get a yes at the moment of payment.
If you cannot obtain approval in the moment, the provider's own instruction is to route the
purchase through Shopify's Shop skill (`https://shop.app/SKILL.md`) instead.

## Flow

1. **Search** — `search_catalog` with `{meta, catalog}`. Pass buyer context
   (`context.address_country`, `context.currency`) for correct pricing and availability.
2. **Resolve** — `lookup_catalog` to resolve several identifiers at once, or `get_product`
   for full detail on one.
3. **Cart** — `create_cart` `{meta, cart}`, then `update_cart` `{meta, cart, id}` and
   `get_cart` `{meta, id}` as the buyer changes their mind.
4. **Checkout** — `create_checkout` `{meta, checkout}`, then `update_checkout`
   `{meta, checkout, id}` to set shipping address and method. `get_checkout` `{meta, id}`
   returns line items, totals, discounts and taxes.
5. **Approve, then complete** — get the human yes, then `complete_checkout`
   `{meta, id, checkout}`. It returns the order ID and the Thank You Page URL.
6. **Track** — `get_order` `{meta, id}`.

## Money — read this before you quote a price

Every price in every response is an **integer in the currency's ISO 4217 minor units**,
paired with a currency code:

```
{"amount": 600,  "currency": "USD"}   ->  $6.00
{"amount": 2500, "currency": "USD"}   -> $25.00
```

Divide by 100 for two-decimal currencies (USD, EUR). Zero-decimal currencies such as JPY are
already whole units — do not divide. Quoting a raw `amount` to a buyer will overstate the
price by 100x.

## Backing out

- Before payment: `cancel_cart` `{meta, id}` and `cancel_checkout` `{meta, id}` both exist
  and are real tools.
- **After `complete_checkout`, there is no reversal in this API.** There is no refund, void
  or cancel-order tool — `get_order` is read only. The only way back is American Eagle's
  human returns process (https://www.ae.com/us/en/content/help/return-policy).
- **No window is published** for `cancel_cart` or `cancel_checkout`. Do not tell a buyer how
  long they have to cancel; the provider does not say.

## Failure handling

- Errors are JSON-RPC 2.0 objects, not RFC 9457 problem details. Read
  `error.data.code` for the machine-readable reason — the HTTP status alone is not enough.
- `-32001` / `invalid_profile_url` means your agent profile URI is missing or unresolvable.
- The endpoint is rate limited per IP. Back off on `429`. No `RateLimit-*` or `Retry-After`
  header is published, so use exponential backoff rather than reading a quota.
- **There is no idempotency key.** Nothing in any `inputSchema` accepts one. A retried
  `complete_checkout` is not deduplicated for you — treat it as unsafe to retry blind, and
  reconcile with `get_order` before re-firing.
- There is no test mode or sandbox. `create_cart` and `create_checkout` create real state
  against the live store.

## Sibling surfaces

`www.toddsnyder.com` is also an AEO brand on the same platform and its `/llms.txt` is
served, but its `/.well-known/*` and `/api/ucp/mcp` sit behind a Cloudflare managed
challenge and could not be verified. Do not assume the same endpoint exists there.
`www.ae.com` and `www.aerie.com` publish **no** commerce API — `www.ae.com/llms.txt` is a
directory of human storefront pages, nothing more.
