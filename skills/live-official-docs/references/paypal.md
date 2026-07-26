# PayPal official sources

Identify sandbox versus live, merchant capability, checkout product, client/server SDK versions, currency, and region. Never retrieve, print, log, or embed credentials while browsing.

| Need | Official URL | Crawl context |
|---|---|---|
| REST platform | https://developer.paypal.com/api/rest/ | Follow only the required auth, idempotency, errors, pagination, and product guide. |
| Orders API | https://developer.paypal.com/docs/api/orders/v2/ | Verify the exact create, confirm, authorize, capture, retrieve, update, or tracking contract. |
| Browser SDK | https://developer.paypal.com/sdk/js/reference/ | Verify the current SDK API, components, callbacks, eligibility, and server boundary. |
| Webhooks | https://developer.paypal.com/api/rest/webhooks/rest/ | Verify subscription, raw event handling, signature verification, retry, and event contract. |
| Production failures | https://developer.paypal.com/api/rest/integration/orders-api/troubleshooting/ | Match exact error names and `debug_id` guidance; do not guess recovery behavior. |

Official domains: `developer.paypal.com`, `api-m.paypal.com`, `paypal.com`, and official `github.com/paypal/*` repositories linked by PayPal.
