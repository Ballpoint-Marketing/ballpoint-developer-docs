# Changelog

Changes to the routes, fields and events documented in this repository.

## Unreleased

- `POST /orders` now checks Hybrid and Greeting letters when the order is created. A missing message, a logo that is not an inline `data:image`, or text the card cannot hold returns `400 INVALID_PRODUCT_CONFIG`; a temporary failure of the check returns `503 LETTER_CHECK_UNAVAILABLE`.
- Hybrid letters fill in the recipient's first name from `{first_name}` or `{{first_name}}` in `message`. Any other tag is rejected with `400 INVALID_PRODUCT_CONFIG`.

## 2026-11-22

- Documented the three letters (`color_letter`, `hybrid_letter`, `greeting_letter`) in a new [Letters](docs/api-reference.md#letters) section: sizes, what is printed and what is written by hand, and the fields each one uses.
- `POST /orders` fields added to the reference and the OpenAPI: `message`, `message_parts`, `logo`, `print_font` and `sender.website`. These fields already existed; only the documentation is new.
- Color Letter artwork is `canvas_json.front` only, at 816 x 1056.
- Production lead time for letters: 6 business days.
- Letters are enabled per account, and cannot be quoted through `POST /v1/billing/orders/preview` yet.

## 2026-11-17

- First release of the developer docs: authentication, quote, order creation, recipient upload, order and tracking reads, print jobs, webhooks, errors and limits.
