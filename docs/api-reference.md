# API reference

Base URLs: `https://staging-api.ballpointmarketing.com` (staging, nothing is mailed) and `https://api.ballpointmarketing.com` (production). Prices in `tcents` use 10,000 units per US dollar.

- [Authentication and access](#authentication-and-access)
- [Quote](#quote)
- [Create an order](#create-an-order)
- [Upload recipients](#upload-recipients)
- [Get an order](#get-an-order)
- [Order tracking](#order-tracking)
- [Print jobs](#print-jobs)
- [Idempotency](#idempotency)
- [Production lead times](#production-lead-times)
- [Not available through the API](#not-available-through-the-api)

## Authentication and access

Every request sends your key in the `X-Partner-Key` header:

```
X-Partner-Key: pk_test_REPLACE_ME
```

Keys are issued by Ballpoint for one account and one environment. A staging key works only against the staging host and a production key only against the production host. The `pk_test_` / `pk_live_` prefix is a label: there is no test mode in production, and an order created with a production key is printed and mailed. Keep the key on your backend only; never put it in a browser or a mobile app.

Each key carries the scopes for the operations it may call. Ballpoint issues one of two profiles:

| Profile | Scopes | Routes |
|---|---|---|
| Orders | `read`, `orders:write`, `recipients:write`, `tracking:read` | [Quote](#quote), [Create an order](#create-an-order), [Upload recipients](#upload-recipients), [Get an order](#get-an-order), [Order tracking](#order-tracking) |
| Print jobs | `read`, `print_jobs:write` | [Print jobs](#print-jobs), [Get an order](#get-an-order) |

Any other route answers `403` with `detail.error.code: "INSUFFICIENT_SCOPE"` and `required_scope: null`, which means the route is not available to that key. See [Errors and limits](errors.md).

## Quote

```
POST /v1/billing/orders/preview
```

Returns the price for a product, postage class and quantity, and runs the same account limit checks as order creation. It creates nothing, so it is also the way to check that a key works.

```bash
curl -X POST https://staging-api.ballpointmarketing.com/v1/billing/orders/preview \
  -H "X-Partner-Key: $BALLPOINT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"product_type": "4x6_printed", "postage_type": "first_class", "piece_count": 500}'
```

```json
{
  "product_type": "4x6_printed",
  "postage_type": "first_class",
  "piece_count": 500,
  "unit_price_tcents": 5388,
  "total_tcents": 2694000,
  "total_dollars": "$269.4000",
  "limits": {
    "passed": true,
    "checks": [
      {"type": "order_limit", "passed": true, "order_cost_cents": 26940, "limit_cents": 500000}
    ]
  }
}
```

`unit_price_tcents` and `total_tcents` are your price. If `limits.passed` is `false`, creating that order would fail with `402`; show the reason before you submit. The response can carry further fields (for example `balance_cents` and `balance_after_cents` on prepaid accounts); ignore any you do not use. The products and postage classes your account can order are agreed during onboarding; the quote rejects any other combination. Letters cannot be quoted through this route yet; your Ballpoint contact gives you their prices.

## Create an order

```
POST /orders
Idempotency-Key: <a new UUID for each new order>
```

Creates one order with its artwork. The order is printed for the recipients you [upload](#upload-recipients) next.

| Field | Required | Description |
|---|---|---|
| `product_id` | Yes | Product identifier, for example `4x6_printed`. |
| `product_type` | Recommended | Canonical product type, for example `4x6_printed`, `6x9_printed`, `color_letter`, `hybrid_letter` or `greeting_letter`. |
| `postage_type` | Recommended | `first_class` or `standard` for printed postcards and Color Letter; `first_class` for Hybrid and Greeting letters. Defaults to `first_class`. |
| `postcard_size` | For postcards | `4x6` or `6x9`. |
| `pieces` | Yes | Number of pieces; must match the number of recipients you upload. |
| `canvas_json` | For postcards and Color Letter | Fabric.js canvas with the artwork. Postcards send `{ "front": …, "back": … }`; Color Letter sends `{ "front": … }` only (see [Letters](#letters)). Images go inline as `data:` URIs (PNG, JPEG or WebP). On postcards Ballpoint adds the address block and postage; do not draw them. |
| `sender` | Recommended | Return address: `name`, `address`, `city`, `state`, `zip`. Printed on postcards and written on the envelope of letters. Hybrid and Greeting letters also print `website`. |
| `message` | For Hybrid and Greeting letters | The letter text. Greeting letters can send `message_parts` instead. See [Letters](#letters). |
| `message_parts` | Optional | Greeting Letter text split into `greeting`, `body` and `signature` (`body` required). If you also send `message`, it must equal the parts joined by a blank line. |
| `logo` | Optional | Logo printed on Hybrid and Greeting letters, as a base64 `data:image` URI (PNG, JPEG, GIF or WebP, up to 450 KB). A link is not accepted. |
| `print_font` | Optional | Handwriting style for the parts written by hand: `cursive` or `printed`. |
| `list_id` | Recommended | Your grouping key (up to 128 characters). Orders that share a `list_id` belong to one Ballpoint campaign. |
| `external_id` | Recommended | Your own reference for the order. |
| `external_user_id` | Optional | Your identifier for the end user the order belongs to; echoed on webhooks. |
| `mail_date` | Optional | `YYYY-MM-DD`. Without it the order goes into production right away; with it the order waits as `scheduled` until its production date (see [lead times](#production-lead-times)). |
| `expected_retail_unit_price_tcents`, `expected_retail_total_price_tcents` | Optional | The quote you showed. If the current price differs, the request fails with `409 PRICE_QUOTE_STALE` and nothing is created. |

A complete request with sample artwork is in [examples/api-onboarding/order.json](../examples/api-onboarding/order.json). Agree the layout of your own design with Ballpoint before the first live order.

Response `200`:

```json
{
  "id": "ord_7f3a2b",
  "external_id": "your-order-123",
  "campaign_id": "camp_9c41",
  "status": "accepted",
  "scheduled_production_date": null,
  "created_at": "2026-03-01T14:00:00Z"
}
```

The response can carry further fields; ignore any you do not use.

The order identifier is **`id`**. Without a `mail_date` the initial status is `accepted`; with a future one it is `scheduled`. Retrying with the same `Idempotency-Key` and body returns the same order (see [Idempotency](#idempotency)).

## Letters

Letters go in an envelope. Ballpoint addresses the envelope by hand and applies the postage, so nothing on the letter itself needs to be left clear for an address or a stamp. None of the three letters has a back side.

| `product_type` | What is printed | What is written by hand | Artwork you send |
|---|---|---|---|
| `color_letter` | One full-color 8.5 x 11 page, folded into a #10 envelope | The envelope address | `canvas_json.front` |
| `hybrid_letter` | A 6.57 x 5 inch card: your logo at the top, the message in a handwriting-style font, your website at the bottom | The envelope address | `message`, `logo`, `sender.website` |
| `greeting_letter` | A 6.57 x 5 inch card: your logo and your website | The message, written in pen after the card is cut, and the envelope address | `message` or `message_parts`, `logo`, `sender.website` |

**Color Letter.** Send the page as `canvas_json.front` with `width: 816` and `height: 1056` (96 canvas units per inch). The canvas is not scaled: a smaller canvas prints in the top-left corner of the page and anything beyond 816 x 1056 is cut. A Color Letter without a non-empty `canvas_json.front` is rejected.

**Hybrid Letter.** The card has a fixed layout and does not use `canvas_json`. Send the letter text in `message`; it is what gets printed. To greet each recipient by name, write `{first_name}` (or `{{first_name}}`) in the message: it is replaced with the recipient's first name, or "Homeowner" when the recipient has none. It is the only tag a Hybrid letter fills in; a message with any other tag, such as `{city}` or `{{last_name}}`, is rejected so that no letter prints with a gap.

**Greeting Letter.** The card has a fixed layout and does not use `canvas_json`. Keep the message to about 350 characters with no paragraph breaks, so it fits the card at the normal handwriting size.

Hybrid and Greeting orders are checked when you create them. A missing message, a logo that is not a readable inline `data:image`, or a message or website that does not fit the card is rejected with `400 INVALID_PRODUCT_CONFIG`, and `error.message` says what to change. If the check is briefly unavailable you get `503 LETTER_CHECK_UNAVAILABLE`; retry later with the same `Idempotency-Key`.

Letters are mailed in a plain white envelope. Letters are enabled per account, and so are decorated envelopes: ask your Ballpoint contact before your first letter order.

On staging, a Color Letter order produces a print file (`render_status` reaches `complete`). Hybrid and Greeting cards are produced at print time, so an order sent as described here, without `canvas_json`, gets no print file on staging. There is no visual proof route; build your preview from the sizes above.

## Upload recipients

```
POST /v1/billing/orders/{order_id}/recipients
Idempotency-Key: <a stable key for this block>
```

```json
{
  "recipients": [
    {
      "first_name": "Jane",
      "last_name": "Doe",
      "company": null,
      "address": "100 Main St",
      "address2": null,
      "city": "San Francisco",
      "state": "CA",
      "zip": "94103",
      "contact_id": "crm-contact-42",
      "address_type": "MAILING"
    }
  ],
  "append": false
}
```

- Up to 10,000 recipients per request. For a larger order, send several blocks with `append: true` after the first one.
- Required per recipient: `address`, `city`, `state` (two letters), `zip` (5 or 5+4 digits), and at least one of `first_name` / `last_name`.
- Optional: `company`, `address2`, `contact_id` (up to 64 characters, your own identifier, returned as sent), `address_type` (`MAILING` or `PROPERTY`).
- `append: false` replaces every recipient already on the order; `append: true` adds to them.

Response `200`:

```json
{
  "order_id": "ord_7f3a2b",
  "accepted": 499,
  "rejected": 1,
  "rejected_details": [{"index": 12, "reason": "At least first_name or last_name is required"}],
  "total_recipients": 499,
  "piece_count": 500,
  "ready": false
}
```

A row missing both names is rejected on its own and the rest are accepted; a malformed required field fails the whole request with `422`. `ready` is `true` when the order has exactly as many recipients as pieces; only then is it rendered. An HTTP `200` alone is not enough: check `rejected` and `ready`.

Recipients can be uploaded while the order is `scheduled`, `accepted` or `prep`. Later the request answers `409 RECIPIENTS_LOCKED`.

| Status | `detail.error.code` | When |
|---|---|---|
| 404 | `ORDER_NOT_FOUND` | The order does not exist or belongs to another account. |
| 409 | `RECIPIENTS_LOCKED` | The order is past `prep`. |
| 400 | `RECIPIENT_COUNT_EXCEEDS_PIECE_COUNT` | More recipients than the order's pieces. |
| 422 | — | Malformed recipient fields. |

**Retries.** Send an `Idempotency-Key` for each block and save it with the exact body. If the response is lost, send the same key and body again: you get the original receipt and nothing is added twice. Reusing a key with a different body returns `422 IDEMPOTENCY_KEY_REUSE`; a block still being processed returns `409 IDEMPOTENCY_KEY_IN_PROGRESS` with `Retry-After`. Receipts are currently kept for 48 hours.

## Get an order

```
GET /v1/billing/orders/{order_id}
```

```json
{
  "id": "ord_7f3a2b",
  "campaign_id": "camp_9c41",
  "product_type": "4x6_printed",
  "postage_type": "first_class",
  "piece_count": 500,
  "production_status": "printing",
  "usps_status": null,
  "display_status": "printing",
  "render_status": "complete",
  "sla_due_at": "2026-03-09T14:00:00Z",
  "status_changed_at": "2026-03-02T09:00:00Z",
  "created_at": "2026-03-01T14:00:00Z",
  "cancelled_at": null
}
```

- `display_status` is the single status to show your users.
- `production_status` follows fulfillment (see the [lifecycle](webhooks.md#order-lifecycle)); `render_status` follows file generation; `usps_status` stays `null` until USPS scans arrive.
- The response can carry further fields; ignore any you do not use.
- An unknown order and another account's order both return `404 ORDER_NOT_FOUND`.

Prefer [webhooks](webhooks.md) to polling.

## Order tracking

```
GET /v1/orders/{order_id}/mail-tracking
```

```json
{
  "mail_status": "delivered",
  "mail_status_label": "Delivered",
  "total_pieces": 500,
  "scanned_pieces": 480,
  "scan_coverage": 0.96,
  "delivered": 420,
  "in_transit": 30,
  "out_for_delivery": 25,
  "rts": 5,
  "forwarded": 0,
  "delivered_rate": 0.84,
  "rts_rate": 0.01,
  "first_scan_at": "2026-03-03T08:15:00Z",
  "last_scan_at": "2026-03-06T14:30:00Z",
  "last_updated_at": "2026-03-06T14:30:00Z"
}
```

| Field | Description |
|---|---|
| `mail_status` | `shipped`, `in_transit`, `out_for_delivery` or `delivered` |
| `scanned_pieces`, `scan_coverage` | Pieces with at least one USPS scan, and their share of the order |
| `delivered`, `delivered_rate` | Pieces with a delivery scan, and their share |
| `rts`, `rts_rate` | Pieces returned to sender (bad addresses), and their share |

Scans appear 1–2 business days after the mail reaches USPS. Once the order is `shipping` or `complete`, the summary starts with `mail_status: shipped` and zero counts until the first scan. Before that, the route answers `404` with `detail.error.code: "ORDER_NOT_MAILED"`, which means "no tracking yet"; read the order status from [Get an order](#get-an-order).

## Print jobs

For finished, print-ready PDFs that you build yourself (for example personalized booklets). Ballpoint prints exactly the file you send: there is no canvas, recipient list or postal layer. Each accepted job becomes one order you can follow with [Get an order](#get-an-order). Print jobs need a key with the print-jobs profile on an account where they are enabled.

| Rule | Value |
|---|---|
| Page size | Every page 5.5 × 8.5 in (396 × 612 pt), no bleed |
| Booklet | Saddle-stitched; `pages_per_booklet` a multiple of 4 from 4 to 32; pad with blank pages |
| Page count | Exactly `booklet_count × pages_per_booklet`, booklets in page order |
| Mailing address | Printed on each booklet's last page with the First Class indicia; no separate recipient list |
| File | Not encrypted; at most 250 MB, 10,000 booklets and 40,000 pages per job |

**1. Request an upload.** `external_id` is your unique ID for the batch (1–128 characters: letters, digits, `.`, `_`, `:`, `-`, starting with a letter or digit).

```bash
curl -X POST https://staging-api.ballpointmarketing.com/v1/print-jobs/upload-url \
  -H "X-Partner-Key: $BALLPOINT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"external_id": "2026-09-25"}'
```

```json
{
  "external_id": "2026-09-25",
  "upload": {
    "method": "POST",
    "url": "https://<bucket>.s3.amazonaws.com/",
    "fields": {"key": "…", "policy": "…", "x-amz-signature": "…"},
    "expires_at": "2026-09-25T15:30:00+00:00",
    "max_bytes": 262144000
  }
}
```

**2. Upload the PDF** with a multipart form POST to `upload.url`: every entry of `upload.fields` first, the file last, before `expires_at` (30 minutes). A file over 250 MB is refused by the storage endpoint itself.

```bash
curl -X POST "<upload.url>" -F "key=<fields.key>" -F "policy=<fields.policy>" … -F "file=@booklets.pdf"
```

**3. Submit the job.**

```bash
curl -X POST https://staging-api.ballpointmarketing.com/v1/print-jobs \
  -H "X-Partner-Key: $BALLPOINT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"external_id": "2026-09-25", "booklet_count": 120, "pages_per_booklet": 8}'
```

`201 Created`:

```json
{
  "order_id": "ord_…",
  "external_id": "2026-09-25",
  "booklet_count": 120,
  "pages_per_booklet": 8,
  "production_status": "accepted"
}
```

Ballpoint validates the file before accepting the job; a rejected job stores nothing, so fix the file, upload it again under the same `external_id` and resubmit. Submitting the same `external_id` with the same counts again returns the existing order (`200`) and never creates a duplicate.

| Status | Code | Meaning |
|---|---|---|
| 403 | `PRINT_JOBS_NOT_ENABLED` | Print jobs are not enabled for this account |
| 409 | `PRINT_JOB_EXISTS` | Upload requested for an `external_id` already submitted |
| 409 | `PRINT_JOB_CONFLICT` | `external_id` already submitted with different counts |
| 409 | `PDF_CHANGED_DURING_SUBMIT` | The upload changed while it was validated; submit again |
| 422 | `PDF_NOT_UPLOADED` | Nothing uploaded for this `external_id` (uploads expire after 7 days) |
| 422 | `PDF_TOO_LARGE` | Over 250 MB |
| 422 | `PDF_UNREADABLE` | Not a readable PDF, or too complex to inspect |
| 422 | `PDF_ENCRYPTED` | Password-protected PDF |
| 422 | `PDF_PAGE_SIZE` | A page is not 5.5 × 8.5 in; the error names the page |
| 422 | `PDF_PAGE_COUNT` | Page total differs from `booklet_count × pages_per_booklet` |
| 400 | `NO_PRICING` | Print-job pricing is not configured for the account yet |
| 503 | `PRINT_JOBS_BUSY` | Another job is being validated; retry after `Retry-After` |
| 503 | `PRINT_JOBS_UNAVAILABLE` | File storage is temporarily unavailable; retry later |

Pricing is per page plus First Class postage per booklet, with a lower postage rate from 500 booklets per job; Ballpoint shares the rates when the account is set up. The price is fixed when the job is accepted.

## Idempotency

`POST /orders` requires an `Idempotency-Key` header, and the recipient upload accepts one.

| Case | Result |
|---|---|
| First request with a key | Processed normally |
| Same key, same body | The original response; nothing is created again |
| Same key, different body | `422 IDEMPOTENCY_KEY_REUSE`; nothing changes |
| A new order or block | Generate a new key |

After a timeout, a `5xx` or a lost response, retry with the **same key and the same body**. Save both before sending, and never generate a new key just because a response was lost.

## Production lead times

With a `mail_date`, Ballpoint schedules production that many business days (Monday to Friday) before it:

| `product_type` | Business days |
|---|---|
| `4x6_printed`, `6x9_printed` | 3 |
| `color_letter`, `hybrid_letter`, `greeting_letter` | 6 |

## Not available through the API

Listing orders, cancelling or rescheduling an order, changing recipients after the order is past `prep`, self-service webhook registration and key creation are not part of this API. Ask your Ballpoint contact for any of them.
