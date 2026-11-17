# Webhooks

Ballpoint notifies your backend when an order changes production status and when its USPS tracking advances.

> Delivery is **at least once**. Your handler must accept duplicates, delays and events out of order.

## Registration

Send your HTTPS endpoint URL to your Ballpoint contact. Ballpoint registers it for your account, subscribes it to the events below and gives you its signing secret. There is no self-service registration.

Your endpoint must:

- accept `POST` with `Content-Type: application/json`;
- use HTTPS and be reachable from the internet;
- answer `2xx` within 10 seconds.

## Events

| Event | When |
|---|---|
| `order.status_changed` | The order's production status changes |
| `order.usps_update` | The order's USPS tracking status advances |

Ignore any other event type you receive. Payload fields sit at the top level of the JSON body, next to `event_id`, `event_type` and `timestamp`. There is no `data` wrapper. Ignore fields you do not use; new fields can be added.

### `order.status_changed`

```json
{
  "order_id": "ord_7f3a2b",
  "campaign_id": "camp_9c41",
  "previous_production_status": "accepted",
  "production_status": "printing",
  "usps_status": null,
  "display_status": "printing",
  "product_type": "4x6_printed",
  "note": null,
  "external_user_id": "user_789",
  "list_id": "marketing_q1_2026",
  "event_id": "b6c3f9d1-2e4a-4d7b-9c1a-5f8e2b3d4c6e",
  "event_type": "order.status_changed",
  "timestamp": "2026-03-01T16:30:00Z"
}
```

- `list_id` and `external_user_id` echo what you sent when you created the order.
- When an order fails for good, the event carries `production_status: "failed"` and an `error_message`.

### `order.usps_update`

```json
{
  "order_id": "ord_7f3a2b",
  "campaign_id": "camp_9c41",
  "previous_usps_status": "shipped",
  "usps_status": "out_for_delivery",
  "production_status": "complete",
  "display_status": "out_for_delivery",
  "product_type": "4x6_printed",
  "external_user_id": "user_789",
  "piece_count": 500,
  "pieces_delivered": 50,
  "pieces_at_destination": 200,
  "pieces_rts": 3,
  "pieces_scanned": 380,
  "event_id": "c7d4a0e2-3f5b-4e8c-8d2b-6a9f3c4e5d7f",
  "event_type": "order.usps_update",
  "timestamp": "2026-03-05T18:30:00Z"
}
```

`usps_status` moves forward only: `shipped` → `out_for_delivery` → `delivered`. This event does not carry `list_id`; join it to your order by `order_id`.

## Headers

```
X-Ballpoint-Signature: sha256=<hex>
X-Ballpoint-Timestamp: 2026-03-01T16:30:00Z
X-Ballpoint-Event: order.status_changed
X-Ballpoint-Event-Id: b6c3f9d1-2e4a-4d7b-9c1a-5f8e2b3d4c6e
X-Ballpoint-Delivery: d4e5f6a7-b8c9-4d0e-1f2a-3b4c5d6e7f8a
```

`X-Ballpoint-Event-Id` identifies the event and stays the same on every retry: use it to deduplicate. `X-Ballpoint-Delivery` changes on every HTTP attempt.

## Verifying the signature

The signature is HMAC-SHA256 with your secret over the timestamp header followed by the **exact raw body bytes**, before any JSON parsing.

Python:

```python
import hmac, hashlib

def verify_signature(body_bytes, timestamp, signature, secret):
    expected = hmac.new(secret.encode(), timestamp.encode() + body_bytes, hashlib.sha256).hexdigest()
    received = signature.removeprefix("sha256=")
    return hmac.compare_digest(expected, received)
```

Node.js:

```javascript
const crypto = require("crypto");

function verifySignature(bodyBuffer, timestamp, signature, secret) {
  const expected = crypto.createHmac("sha256", secret)
    .update(timestamp, "utf8")
    .update(bodyBuffer)
    .digest();
  const received = Buffer.from(signature.replace("sha256=", ""), "hex");
  return received.length === expected.length && crypto.timingSafeEqual(expected, received);
}
```

Before acting on an event:

1. Verify the signature with a constant-time comparison.
2. Reject timestamps more than 5 minutes old or more than 2 minutes in the future.
3. Reject an `X-Ballpoint-Event-Id` you already processed (keep processed IDs for at least 24 hours).
4. Answer `2xx` quickly and do the work asynchronously.

Working receivers: [Express](../examples/express-integration/), [AWS Lambda](../examples/lambda-webhook/) (DynamoDB deduplication), [Cloudflare Worker](../examples/cloudflare-worker-webhook/) (KV deduplication). Any store with a unique key on the event ID works.

## Retries

- Each round makes up to 3 HTTP attempts, 1 and 2 seconds apart, each with a 10-second timeout.
- A failed event is retried in up to 5 rounds per endpoint, 10 s, 30 s, 90 s and 270 s apart.
- A `4xx` answer ends the current round early but does not stop later rounds.
- After 5 failed rounds the event is not retried again; there is no automatic replay, so ask Ballpoint to resend it if you need it.
- After 10 consecutive failed deliveries the endpoint is disabled and Ballpoint contacts you.

## Order lifecycle

`production_status` moves forward only:

```
Printed postcards:     [scheduled →] accepted → prep → printing → stamping → shipping → complete
Handwritten postcards: [scheduled →] accepted → prep → printing → writing → stamping → shipping → complete
```

| Status | Meaning |
|---|---|
| `scheduled` | Waiting for its production date (orders with a future `mail_date`) |
| `accepted` | Queued for production |
| `prep` to `shipping` | In production |
| `complete` | Dropped at USPS; first scans arrive 1–2 business days later |
| `cancelled` | Cancelled before production |
| `failed` | Ended with an unrecoverable error |

`usps_status` is set from USPS scans: `shipped` (at least one piece scanned), `out_for_delivery` (51% of pieces at the destination facility) and `delivered` (80% of pieces delivered or returned). Show `display_status` to your users; it combines both.

## Testing

Staging orders are never mailed, so their status does not advance on its own. Ask your Ballpoint contact to trigger signed test events against your endpoint, and check a valid event, a duplicate and a tampered body.
