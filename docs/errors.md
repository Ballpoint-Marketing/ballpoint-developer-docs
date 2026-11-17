# Errors and limits

## Error format

Errors return a JSON object with the details under `detail.error`:

```json
{
  "detail": {
    "error": {
      "type": "validation_error",
      "code": "INVALID_PRODUCT_CONFIG",
      "message": "Unknown product_type 'magic_letter'",
      "trace_id": "tr_9f2a1c7b4e6d0a13"
    }
  }
}
```

| Field | Description |
|---|---|
| `type` | Error class, for example `validation_error`, `authentication_error`, `authorization_error`, `not_found`, `conflict`, `rate_limit_error`, `billing_error`, `server_error` |
| `code` | Machine-readable code. Branch on this one. |
| `message` | Human-readable explanation, for logs |
| `trace_id` | Quote it in support requests |

Some codes add context, for example `required_scope` on `INSUFFICIENT_SCOPE`. A request that fails validation before it reaches the route (a missing field, a wrong type) answers `422` with `detail` as a list of `{"loc", "msg", "type"}` entries instead.

## Status codes

| Status | Meaning | Retry? |
|---|---|---|
| `400` | Malformed request or a product the account cannot order | No; fix the request |
| `401` | Missing or invalid key | No; check the key and the host |
| `402` | Not enough prepaid balance, or the account limit was reached | No; see [Prepaid balance](#prepaid-balance) |
| `403` | The key cannot use this route (`INSUFFICIENT_SCOPE`) or the account is inactive | No; contact Ballpoint |
| `404` | Not found, or it belongs to another account | No |
| `409` | Conflict, for example a stale price or an idempotency key still in progress | Depends on the code |
| `422` | Validation error, or an idempotency key reused with a different body | No; fix the request |
| `429` | Rate limit | Yes, after `Retry-After` |
| `500`, `502`, `504` | Server or gateway error | Yes, with the same idempotency key |
| `503` | Temporarily unavailable | Yes, after `Retry-After` |

Codes you are likely to meet:

| Code | What to do |
|---|---|
| `INVALID_PARTNER_KEY` | `401`: the key is missing, wrong or for the other environment. |
| `IDEMPOTENCY_KEY_REQUIRED` | `400`: `POST /orders` was sent without an `Idempotency-Key`. |
| `ORDER_LIMIT_EXCEEDED`, `DAILY_LIMIT_EXCEEDED`, `MONTHLY_LIMIT_EXCEEDED` | `402`: the order exceeds a spending limit agreed for your account. The quote reports the same checks in `limits`. |
| `INSUFFICIENT_SCOPE` | `required_scope: null` means the route is not available to your key. Ask Ballpoint. |
| `NO_PRICING` | The product or postage is not priced for your account. Do not proceed with a guessed price. |
| `PRICE_QUOTE_STALE` | The price changed after your quote. Get a new quote, show it, then create the order with a new idempotency key. |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | Wait for `Retry-After`, then send the same request with the same key. |
| `IDEMPOTENCY_KEY_REUSE` | The key was reused with a different body. Recover the saved request; do not create another order blindly. |
| `ORDER_NOT_FOUND` | Check the ID, the environment and the account. |
| `ORDER_NOT_MAILED` | No tracking yet; the order has not been mailed. |
| `RECIPIENTS_LOCKED` | The order is too far into production to change its recipients. |

## Prepaid balance

Some accounts pay in advance. Each order is debited from the balance when it is created.

- The first deposit (US$10 minimum) is paid by card through a Ballpoint payment link, and the card is saved.
- When an order costs more than the balance, the saved card is charged first: the shortfall, or US$50 if the shortfall is smaller. The order is then created as usual.
- Cancelled orders are credited back to the balance. Unused balance is not refunded.

| Status | Code | Meaning | What to do |
|---|---|---|---|
| `402` | `INSUFFICIENT_BALANCE` | The balance does not cover the order and no card is saved | Pay a deposit link, then retry |
| `402` | `AUTO_REFILL_FAILED` | The saved card was declined or needs confirmation | Update the card or pay a deposit link, then retry with the same `Idempotency-Key` |
| `503` | `PAYMENT_PROVIDER_UNAVAILABLE` | The card refill could not reach the payment provider | Retry with the same `Idempotency-Key` after `Retry-After` |

No order is created and nothing is debited when any of these is returned. A retry with the same key never charges the card twice for the same refill.

## Retry strategy

```
Attempt 1: immediately
Attempt 2: after 1 s
Attempt 3: after 2 s
Attempt 4: after 4 s
Attempt 5: after 8 s, then stop and investigate
```

Always honour `Retry-After` when it is present, and retry `POST /orders` and recipient uploads with the same idempotency key and body.

## Rate limits

- 60 requests per minute per key;
- 120 requests per minute per account, across its keys;
- a daily request cap per account, 10,000 requests a day unless agreed otherwise.

Rate-limited responses carry `X-RateLimit-Limit` and `X-RateLimit-Remaining` for the per-key window. An account limit (`ACCOUNT_RPM_EXCEEDED`, `ACCOUNT_RPD_EXCEEDED`) can answer `429` while the per-key counter still has room, and a daily-limit `Retry-After` can last until midnight UTC.
