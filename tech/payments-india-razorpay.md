# Payments — India, Razorpay

This org is **Razorpay-only for India**. Stripe is not part of the stack — never
introduce Stripe SDKs, keys, or webhooks into any India-facing payment flow.

## Applies when
- Deps on the `razorpay` SDK.
- Env var names `RAZORPAY_KEY_ID` / `RAZORPAY_KEY_SECRET` / `RAZORPAY_WEBHOOK_SECRET` in code or config.
- A webhook handler reads `X-Razorpay-Signature`.

## Authoritative sources
| Need | URL |
|---|---|
| Product/integration docs | https://razorpay.com/docs/ |
| REST API reference | https://razorpay.com/docs/api/ |
| API changelog | https://razorpay.com/docs/api/changelog/ |
| Webhooks overview | https://razorpay.com/docs/webhooks/ |
| Webhook validate/test | https://razorpay.com/docs/webhooks/validate-test/ |
| Route (split payments) | https://razorpay.com/docs/payments/route/ |
| Settlements | https://razorpay.com/docs/payments/settlements/ |
| MCP server | https://razorpay.com/docs/mcp-server/ |
| MCP server (GitHub) | https://github.com/razorpay/razorpay-mcp-server |

## Non-obvious rules
- **Amounts are always integer paise.** ₹499.00 is `49900` — a rupee float or missed ×100 is the most common first-integration bug.
- **Order creation is mandatory before checkout.** The server creates the Order from authoritative data; the client only ever receives `order_id` — amount can never be client-tampered.
- **Webhook signature uses a separate secret**, never the API key secret. Verify HMAC-SHA256 over the raw, unparsed body against `X-Razorpay-Signature`; re-serializing breaks the signature.
- **Webhook delivery is at-least-once, can arrive out of order or duplicated.** Handlers must be idempotent on event id or (payment/order id + type).
- **`authorized` is not `captured`.** An uncaptured authorization auto-refunds within the window — set `payment_capture: 1` at order creation or capture explicitly.
- **Idempotency-Key header prevents duplicate mutations on retry** — required on every mutating call from a retry-capable client.
- **Refunds are asynchronous.** A successful create-refund response means "initiated," not "returned" — confirm via webhook or a subsequent fetch.
- **Route linked accounts need completed KYC/activation** before receiving a transfer. Platform fee is set on the `transfers` array at creation, not deducted after.
- **Settlements are per-account, not lumped** — standard cycle T+2 working days; Route settlements reconcile separately from the platform's own.
- **PCI boundary: use hosted Checkout.** Raw card PAN/CVV must never touch app servers or logs — a custom card form pulls the app into PCI-DSS scope.
- **Test/live modes use entirely separate key pairs and webhook secrets** — a test secret never validates a live signature.

## Production checklist
- [ ] All amounts stored/transmitted as integer paise via one conversion function
- [ ] Server creates the Order; client never sends a raw amount to checkout
- [ ] Webhook handler verifies HMAC-SHA256 over raw body with the webhook secret
- [ ] Webhook handler idempotent against duplicate/out-of-order delivery
- [ ] `Idempotency-Key` sent on every mutating retry-capable call
- [ ] Capture flow explicit and tested against the auto-refund-on-timeout window
- [ ] Refund state confirmed via webhook/fetch, never assumed from the create response
- [ ] Route: linked-account activation checked before first transfer
- [ ] Test/live keys and webhook secrets under distinctly named env vars
- [ ] No card PAN/CVV ever logged, stored, or transits the app server

## Never
- Never introduce Stripe, or any gateway other than Razorpay, into an India flow.
- Never trust a client-supplied amount, currency, or order_id at capture time.
- Never verify a webhook signature against a parsed/re-serialized body.
- Never treat a create-refund `200` as proof the customer received their money.
- Never store the webhook secret and API key secret interchangeably.
- Never build a raw card-number input that posts to the application's own server.
