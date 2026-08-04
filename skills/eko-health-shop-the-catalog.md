---
name: Shop the Eko Health catalog
description: >-
  Search Eko Health's stethoscope and accessory catalog, look up a product and its
  variants, build a cart, and hand a checkout URL back to the buyer — using the live
  MCP server Eko Health publishes at www.ekohealth.com/api/mcp, and obeying the
  human-approval rule Eko states in its own agents.md.
api: mcp/eko-health-mcp.yml
transport: https://www.ekohealth.com/api/mcp
operations:
  - search_catalog
  - get_product_details
  - get_cart
  - update_cart
  - search_shop_policies_and_faqs
generated: '2026-08-04'
method: generated
source: mcp/eko-health-mcp-tools.json (verbatim tools/list, HTTP 200, 2026-08-04)
---

# Shop the Eko Health catalog

Eko Health serves a Model Context Protocol server from its own host. `tools/list` is
anonymous — no credentials are needed to browse, search, or build a cart. Every tool
name and every parameter below was read from the live `tools/list` response saved at
`mcp/eko-health-mcp-tools.json`; nothing here is invented.

## Endpoint

```
POST https://www.ekohealth.com/api/mcp
Content-Type: application/json
Accept: application/json, text/event-stream
```

JSON-RPC 2.0. Errors come back as `{"jsonrpc":"2.0","id":…,"error":{"code":…,"message":…,"data":…}}`.

## Hard rule — checkout is for humans

Eko's own `agents.md` and `robots.txt` state it: *"Checkouts are for humans. Do NOT
complete checkout, payment, or order placement automatically — no scripted form fills,
browser automation, or end-to-end agent flows that finalize payment without an explicit,
contemporaneous human approval step."* Build the cart, then hand the buyer the checkout
URL. Do not finalize payment.

## Steps

1. **Answer policy questions first, if that is what was asked.**
   Call `search_shop_policies_and_faqs` with `{"query": "<the buyer's question>"}`
   (`query` is the only required field; `context` is optional free text about the buyer).
   This covers return policy, shipping policy, hours, and contact details.

2. **Find candidate products.**
   Call `search_catalog`. Parameters live under the `catalog` object — supply at least
   one of `catalog.query` (free text) or filter criteria. Use `catalog.context` to
   localize: `address_country` (ISO 3166-1 alpha-2), `address_region`, `postal_code`,
   `language` (BCP 47), `currency` (ISO 4217).
   Results are paginated: pass the `pagination.cursor` from the response back on the
   next call when the buyer asks for more.

3. **Confirm the exact variant.**
   Call `get_product_details` with `product_id` (required, a Shopify global ID of the
   form `gid://shopify/Product/123`). Pass `options` to pin a variant, e.g.
   `{"Color": "Black"}` — option matching is case-insensitive and partial options are
   accepted. Without `options` you get the first available variant. `country` and
   `language` localize the result.

4. **Build the cart.**
   Call `update_cart`. It is a single consolidated write: `add_items`, `update_items`,
   `remove_line_ids`, `buyer_identity`, `delivery_addresses_to_add`,
   `delivery_addresses_to_replace`, `selected_delivery_options`, `discount_codes`,
   `gift_card_codes`, `note`. Shipping options only become available once a delivery
   address is on the cart, so add the address before asking for shipping choices.

5. **Read the cart back and hand off.**
   Call `get_cart` with `cart_id`. The response carries the line items, shipping
   options, discount info, and the **checkout URL**. Give the buyer that URL and stop.

## What this skill does not cover

This is Eko Health's **commerce** surface. It has no access to patient recordings,
auscultation audio, ECG data, or the Eko AI algorithms. Those live behind the Eko
Connect Enterprise SDK and the auth-gated REST API at `api.ekodevices.com`, which
publishes no OpenAPI and is licensed through Eko sales — see
`apis.yml` → *Eko Connect Enterprise SDK and API*.

## Related artifacts

- `mcp/eko-health-mcp.yml` — server manifest, plus the gated `/api/ucp/mcp` endpoint
- `mcp/eko-health-mcp-tools.json` — verbatim `tools/list` with full input schemas
- `conventions/eko-health-conventions.yml` — pagination, identifiers, error envelope
- `authentication/eko-health-authentication.yml` — OAuth/OIDC for customer-scoped calls
- `llms/eko-health-agents.md` — Eko's own agent instructions
