---
name: glowbar-cart-and-checkout
description: Build a cart and drive a Glowbar checkout to the point of payment over the store's UCP/MCP endpoint, stopping at the mandatory human buyer-approval step.
api: Glowbar UCP Commerce MCP API
endpoint: https://glowbar.com/api/ucp/mcp
operations:
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-22'
method: generated
source: mcp/glowbar-mcp-tools.json (live tools/list, HTTP 200, 2026-08-22) + https://glowbar.com/robots.txt
---

# Buy from Glowbar

This flow ends in a real charge. Read the stop condition first.

## Stop condition — non-negotiable

Glowbar publishes this in `robots.txt`, `agents.md` and `llms.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent flows that
> finalize payment without an explicit, contemporaneous human approval step.

You may do everything up to and including pricing the order. You may **not** call
`complete_checkout` without the buyer approving that specific amount, at that moment. If you
cannot obtain contemporaneous approval, do not proceed — route the buyer through Shopify's Shop
skill (https://shop.app/SKILL.md), which the store names as the sanctioned alternative.

## Steps

1. **Find the items** — see the `glowbar-catalog-search` skill.

2. **Create the cart** — `create_cart` with the chosen variants. Remember `meta.ucp-agent.profile`
   on every call.

3. **Adjust** — `update_cart` (with the cart `id`) to change quantities or lines; `get_cart` to
   read current contents. Both are freely reversible; re-issue `update_cart` to undo. If the
   buyer abandons, call `cancel_cart`.

4. **Create the checkout** — `create_checkout`. This charges nothing. It returns line items,
   totals, discounts and taxes.

5. **Set fulfillment** — `update_checkout` with the shipping address and method. Note the store's
   declared constraints: **no multi-destination shipping**, and the only allowed method
   combination is `[shipping]`. This call carries buyer PII — collect it, do not guess it.

6. **Read the real total back to the buyer** — `get_checkout`. Convert minor units to major
   units before quoting (`{"amount": 2500, "currency": "USD"}` is $25.00). This is the rehearsal
   step: there is no dry-run flag, so the priced checkout is your only preview of the final
   amount.

7. **Get approval.** Explicit, at this moment, for this total. Then and only then:

8. **Complete** — `complete_checkout`. Its schema **requires** `meta.idempotency-key` alongside
   `meta.ucp-agent`. Generate one key per purchase intent and reuse it on any retry. Never
   generate a fresh key to retry a call that may have succeeded — that is how an agent
   double-charges a buyer.

9. **Confirm** — the result carries the order ID, a Thank You Page URL, or errors encountered.
   Errors come back **in the result**, not as a JSON-RPC error, so check the payload even on a
   200. Use `get_order` to read the order afterwards.

## Reversibility — know this before step 8

| Action | Undo | Window |
|---|---|---|
| `create_cart` / `update_cart` | `update_cart`, `cancel_cart` | any time before checkout |
| `create_checkout` / `update_checkout` | `cancel_checkout` | any time before `complete_checkout` |
| `complete_checkout` | **none via the API** | see below |

There is no refund, void or reverse tool. Once `complete_checkout` succeeds, the only remedy is
Glowbar's human returns process: *"We can accept unopened product for an exchange or store credit
within 21 days of purchase"* (https://glowbar.com/policies/refund-policy). That is **exchange or
store credit, not a refund to the original payment method**, and only for **unopened** product.

Note a live inconsistency in Glowbar's own pages: the refund policy says 21 days, the FAQ
(https://glowbar.com/pages/faqs) says 14. Quote the shorter one to a buyer if you must promise
anything.

Facial services and membership payments are **final sale** and cannot be returned at all.

## Errors

- `-32001` / `invalid_profile_url` (HTTP 422) — missing or unfetchable `meta.ucp-agent.profile`,
  or an unknown method name.
- `429` — per-IP rate limit. No `Retry-After` is returned; back off exponentially.
