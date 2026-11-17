# Ballpoint Marketing API — Developer Docs

Send printed direct mail from your own backend: quote, create an order with your artwork, upload the mailing list, follow production and USPS delivery, and receive signed status notifications.

Accounts and API keys are set up by Ballpoint. There is no self-service signup; contact your Ballpoint representative to get a staging account.

## Start here

**[START_HERE.md](START_HERE.md)** walks through a complete staging order in eight steps: check access, quote, create the order with sample artwork, upload two recipients, read the order and receive a signed webhook. The last steps need Ballpoint: checking the print files and sending test webhooks.

## Documentation

| Page | What it covers |
|------|----------------|
| [API reference](docs/api-reference.md) | Authentication, the routes your key can call, request and response fields, idempotency |
| [Webhooks](docs/webhooks.md) | Registration, events, signature verification, retries and deduplication |
| [Errors and limits](docs/errors.md) | Error format, status codes, prepaid balance, retries, rate limits |
| [Going live](docs/going-live.md) | Staging vs. production, go-live checklist, support |
| [OpenAPI](openapi.yaml) | Machine-readable specification of the same routes |
| [Changelog](CHANGELOG.md) | Changes to the routes and events documented here |

## Examples

| Example | Description |
|---------|-------------|
| [Postman collection](postman/ballpoint.postman_collection.json) | The routes in this guide, with staging and production environments |
| [Onboarding fixtures](examples/api-onboarding/) | Sample order (with artwork), quote and recipient files used by START_HERE |
| [Express](examples/express-integration/) | Node.js/Express: create a staging order and receive webhooks |
| [AWS Lambda](examples/lambda-webhook/) | Webhook receiver with signature verification and deduplication |
| [Cloudflare Worker](examples/cloudflare-worker-webhook/) | Webhook receiver on Workers with KV deduplication |

## Environments

| | Base URL | Mail printed and sent |
|---|---|---|
| Staging | `https://staging-api.ballpointmarketing.com` | No |
| Production | `https://api.ballpointmarketing.com` | Yes |

Every request carries your key in the `X-Partner-Key` header. The environment decides whether mail goes out, never the key prefix.

## Security

See [SECURITY.md](SECURITY.md) to report a vulnerability. Never include keys or webhook secrets in support requests or public issues.
