# Payments — PayPal

## Applies when
- A manifest lists `@paypal/paypal-server-sdk` or `paypal-rest-sdk`.
- A `services/paypal` directory or a `paypal.ts`/`paypal.php` server module
  exists.

## Authoritative sources
| Need | URL |
|---|---|
| Developer docs | https://developer.paypal.com/docs/ |
| SDKs/API (GitHub org) | https://github.com/paypal |

## Non-obvious rules
- **Sandbox and live are entirely separate credential sets** (client
  id/secret, webhook ids) tied to separate PayPal developer accounts — a
  sandbox secret doesn't just fail loudly against live endpoints, it can
  silently target the wrong environment. Verify the base host
  (`api-m.sandbox.paypal.com` vs `api-m.paypal.com`) matches the credential
  pair in use before trusting any test result.
- **Webhook verification is NOT an HMAC-over-raw-body check like Razorpay's.**
  PayPal requires calling `/v1/notifications/verify-webhook-signature` (or the
  SDK equivalent) with the transmission headers
  (`PAYPAL-TRANSMISSION-ID`, `PAYPAL-TRANSMISSION-TIME`, `PAYPAL-CERT-URL`,
  `PAYPAL-AUTH-ALGO`, `PAYPAL-TRANSMISSION-SIG`) plus the registered webhook
  id — there is no local secret to HMAC against, and skipping the API call in
  favor of a local signature check is a no-op that always "passes."
- **Orders and Subscriptions are different lifecycles.** The Orders API
  (`Order`/`Capture`, one-time) and the Subscriptions API
  (`Subscription`/`Billing Plan`, recurring) have different object graphs and
  webhook event sets — do not reuse order-capture idempotency or status logic
  for subscription events.
- **Idempotency requires the `PayPal-Request-Id` header** on create-order and
  create-capture calls — omitting it on a client/network retry can produce a
  duplicate charge.
- **Amounts are decimal strings in major units** (`"19.99"`), not integer
  minor units like Razorpay's paise. Decimal precision is currency-specific
  (e.g., JPY has 0 decimal places) — never hardcode 2 decimal places across
  all currencies.
- **Renaming a `stripe/` directory to `paypal/` is not a migration.** Verified
  live hazard: handlers under a `paypal`-named path can still import the
  Stripe SDK, call Stripe endpoints, and query Stripe-shaped tables. Diff the
  actual imports and outbound API calls — never trust the directory name as
  evidence of which gateway is really in use.
- Multiple gateways can be live in the same codebase at once (Stripe
  mid-migration alongside a live PayPal integration) — confirm which gateway
  a given handler actually calls before editing or trusting it.

## Production checklist
- [ ] Sandbox and live client id/secret pairs stored under distinctly named
      env vars, never shared or defaulted into each other
- [ ] Webhook signature verified via PayPal's verify-webhook-signature call,
      never a local/home-grown HMAC
- [ ] `PayPal-Request-Id` idempotency header sent on every mutating create call
- [ ] Currency decimal precision handled per-currency, not hardcoded
- [ ] Order and Subscription lifecycles handled by distinct, non-shared code
      paths
- [ ] Any module renamed toward `paypal` audited import-by-import for
      leftover Stripe SDK calls or Stripe table/collection queries
- [ ] Webhook handler idempotent against at-least-once, possibly out-of-order
      delivery

## Never
- Never assume a directory or file named `paypal` is free of Stripe SDK calls
  — verify by reading the actual imports.
- Never trust a webhook payload without calling PayPal's
  verify-webhook-signature check first.
- Never reuse a sandbox credential pair against a live endpoint, or vice
  versa.
- Never disable TLS verification on outbound calls to PayPal's API.
- Never ship a simulated/mock payment success path reachable in production.
