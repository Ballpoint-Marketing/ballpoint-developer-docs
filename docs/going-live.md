# Going live

## Staging and production

| | Staging | Production |
|---|---|---|
| Base URL | `https://staging-api.ballpointmarketing.com` | `https://api.ballpointmarketing.com` |
| Key | Staging key | Production key |
| Orders created | Yes | Yes |
| Mail printed and sent | No | **Yes** |
| USPS tracking | No | Yes, 1–2 business days after the drop |
| Charges | None | According to your agreement |
| Validation and errors | Same as production | — |

Staging runs the same code and validation as production with its own data and keys. There is no test mode in production: any order created there is printed and mailed. Staging orders keep their initial status, because nothing is produced; ask Ballpoint to trigger test webhooks.

## Checklist

- [ ] The full [START_HERE](../START_HERE.md) run passes on staging.
- [ ] Quotes match the products, postage and prices agreed with Ballpoint.
- [ ] Order creation and recipient uploads retry with the same idempotency key and body; no retry creates a second order.
- [ ] Your handler checks `rejected` and `ready` after every recipient upload.
- [ ] Your webhook receiver verifies signatures, rejects old or future timestamps, deduplicates on `X-Ballpoint-Event-Id` and answers quickly.
- [ ] `429` waits for `Retry-After`; `5xx` retries with the same idempotency key.
- [ ] Ballpoint has checked the printed output of your artwork on staging.
- [ ] Production key, production webhook endpoint and billing are set up with Ballpoint.
- [ ] The first production order is small and watched end to end.

## Support

Ballpoint shares the support contact during onboarding. When you report a problem, include the method and path, HTTP status, `detail.error.code`, `trace_id` and order ID. Never include the API key or the webhook secret.

Service status:

```bash
curl https://api.ballpointmarketing.com/health
```

`/health` only shows that the service is up; it does not check your key. Use the [quote](api-reference.md#quote) for that.

Urgent: order creation failing for many requests, webhooks stopped, or every request answering `401`.
