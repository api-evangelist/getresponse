---
name: getresponse-ecommerce-sync
description: >
  Use this skill to sync an ecommerce catalog and order history into GetResponse so that
  abandoned-cart recovery, product recommendations and purchase-triggered automation can run.
  Triggers: whenever the user wants to (1) register a store with GetResponse, (2) push products,
  variants, categories or taxes, (3) record a cart so abandoned-cart automation fires, or
  (4) record an order against a known contact.
  Do NOT use for sending newsletters (use getresponse-newsletter-skill), landing pages,
  webinars, or transactional email.
api: openapi/getresponse-shops-openapi.yml
operations:
  - createShop
  - getShopList
  - createProduct
  - createProductVariant
  - createCategory
  - upsertProductCategories
  - createCart
  - updateCart
  - createOrder
  - getOrderList
---

# GetResponse ecommerce sync

Grounded in the provider-published OpenAPI at
`https://apireference.getresponse.com/open-api.json`. Every operationId below is verified
present in that spec.

## Before you start

Read `conventions/getresponse-conventions.yml` first. Three things will bite you otherwise:

- **Auth header.** `X-Auth-Token: api-key <KEY>` — the literal `api-key ` prefix is required.
  GetResponse MAX accounts must also send `X-Domain: <account-domain>`.
- **No idempotency.** There is no `Idempotency-Key`. Every write below is unsafe to blind-retry.
  Deduplicate on your side using `externalId` before calling, and treat error code **1008**
  (HTTP 409, "duplicate unique property") as "already exists" rather than as a failure.
- **Percent-encode bracketed query keys.** `query[externalId]` must go on the wire as
  `query%5BexternalId%5D`. Unencoded brackets are the most common cause of a spurious HTTP 400.

## Step 1 — Resolve or create the shop

1. `getShopList` (`GET /shops`) — look for an existing shop. Filter with
   `query%5Bname%5D=<name>` rather than paging the whole list.
2. If none matches, `createShop` (`POST /shops`) with the store name, currency and locale.
3. Keep the returned `shopId`. Every subsequent call in this skill is nested under it.

Do **not** create a second shop for a store that already exists — GetResponse will happily
accept it and you will split the customer's ecommerce reporting in two.

## Step 2 — Push the catalog

For each product:

1. `createProduct` (`POST /shops/{shopId}/products`). Set `externalId` to your own primary key —
   this is the only join key you get back.
2. `createProductVariant` (`POST /shops/{shopId}/products/{productId}/variants`) for each SKU.
   A product with no variant cannot be referenced by a cart or order line.
3. `createCategory` (`POST /shops/{shopId}/categories`) once per category, then
   `upsertProductCategories` (`POST /shops/{shopId}/products/{productId}/categories`) to bind
   them. `upsert` here means the call is safe to repeat — prefer it over creating links twice.

Throughput: the account budget is 30,000 calls per 10-minute frame, 80/second, max 10 in
flight. Read `X-RateLimit-Remaining` from each response and stop when it approaches zero; on a
429 wait `context.timeToReset` seconds. Do not open more than 10 concurrent connections.

## Step 3 — Record carts (this is what makes abandoned-cart work)

`createCart` (`POST /shops/{shopId}/carts`) with `contactId`, `externalId` and the line items
referencing `productId` + `variantId`. Update it with `updateCart`
(`POST /shops/{shopId}/carts/{cartId}`) as the shopper changes it.

The contact must already exist in GetResponse. If you only have an email address, resolve it
first with `getContactList` (`GET /contacts?query%5Bemail%5D=…`) — do not create the cart
against a guessed id.

Delete the cart with `deleteCart` when it converts, otherwise abandoned-cart automation will
keep chasing a customer who already bought.

## Step 4 — Record orders

`createOrder` (`POST /shops/{shopId}/orders`) with `contactId`, `externalId`, `totalPrice`,
`currency` and the selected variants. Pass the originating `cartId` so GetResponse can attribute
the conversion. Reconcile with `getOrderList` (`GET /shops/{shopId}/orders`) filtered on
`query%5BexternalId%5D`.

## Error handling

Branch on the numeric `code` in the body, not on the HTTP status — one 400 carries a dozen
distinct codes. Full registry in `errors/getresponse-error-codes.yml`. The ones you will meet:

| code | status | what to do |
|---|---|---|
| 1001 | 400 | A referenced id (shop, product, contact) does not exist — resolve it first |
| 1003 | 400 | Format validation — usually an unencoded bracket or a bad date |
| 1008 | 409 | Duplicate `externalId` — treat as success, look the resource up |
| 1013 | 404 | Id not found |
| 1014 | 403 | Auth failure — note this is **403**, not 401 |
| 1015 | 429 | Rate limited — wait `context.timeToReset` seconds |
| 1027 | 400 | Contact limit reached — a plan ceiling, not retryable |

Every error body carries a `uuid`. Log it; it is the only request identifier GetResponse gives
you, and support will ask for it.

## Confirmations required

Creating a shop, and deleting any product, cart or order, changes what the customer's live
automation will do. Confirm with the user before any `delete*` call. Never delete a shop.
