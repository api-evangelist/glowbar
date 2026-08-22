---
name: glowbar-catalog-search
description: Search and inspect the Glowbar skincare product catalog over the store's public UCP/MCP endpoint, with correct buyer context, pagination and price conversion.
api: Glowbar UCP Commerce MCP API
endpoint: https://glowbar.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
generated: '2026-08-22'
method: generated
source: mcp/glowbar-mcp-tools.json (live tools/list, HTTP 200, 2026-08-22)
---

# Search the Glowbar catalog

Read-only. Nothing in this skill creates a cart, a checkout, or a charge.

## Before you call anything

Every tool on this endpoint requires an agent profile URI. Put it in `meta` on **every** call:

```json
{ "meta": { "ucp-agent": { "profile": "https://your-agent.example/profile.json" } } }
```

Omit it and the endpoint answers HTTP 422 with JSON-RPC error `-32001`, `code: invalid_profile_url`.
The same error is returned for an unknown method name, so if you get it unexpectedly, check your
method spelling before you blame the profile.

## Steps

1. **Search** — call `search_catalog` with `catalog.query` set to the buyer's intent.
   Always pass `catalog.context.address_country` (ISO 3166-1 alpha-2) and
   `catalog.context.currency` (ISO 4217); the store's own instructions say pricing and
   availability are wrong without them. Optional narrowing lives in `catalog.filters`
   (`categories`, `price.min` / `price.max` — both in **minor units**).

2. **Page** — results are cursor-paginated. `catalog.pagination.limit` defaults to `10`
   (minimum `1`). To get more, pass the `pagination.cursor` from the previous response back in
   `catalog.pagination.cursor`. Do not page speculatively — the tool description says to fetch
   additional pages only "when users request more results".

3. **Look up specifics** — `lookup_catalog` resolves several products or variants by identifier
   in one call; `get_product` returns one complete product. Prefer `lookup_catalog` over a loop
   of `get_product` calls.

4. **Convert prices before you speak them.** Every amount is an integer in ISO 4217 **minor
   units** paired with a currency code: `{"amount": 5400, "currency": "USD"}` is **$54.00**.
   Divide by 100 for two-decimal currencies; JPY and other zero-decimal currencies are already
   whole units. Quoting the raw integer to a buyer is a 100x error.

## Rules

- Back off on `429`. The endpoint is rate-limited per IP and returns no `Retry-After`, so use
  exponential backoff of your own.
- Identifiers are Shopify global IDs, shaped `gid://shopify/{Type}/{id}`. Pass them back
  verbatim; never construct one.
- The catalog is small — 30 products at last check, ranging $10.00 to $195.00 — so a broad
  search plus client-side filtering is usually better than many narrow searches.
- Facial services and memberships are **not** in this catalog. They are sold in-studio and
  through a separate booking system at https://bookings.glowbar.com, which has no API. If the
  buyer wants an appointment, hand them that URL; do not try to book it.
