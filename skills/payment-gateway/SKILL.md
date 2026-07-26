---
name: payment-gateway
description: Use when implementing or reviewing checkout, billing, payments, subscriptions, refunds, offers, UPI, wallets, saved methods, international payments, Razorpay, or PayPal integrations.
---

# Production payment gateway

**REQUIRED LIVE REFERENCE:** Use `live-official-docs`, then read only its Razorpay and/or PayPal map before coding. Resolve current SDK and API syntax from the repository plus those official pages.

## Scope

Support only **Razorpay and PayPal**, both international-enabled. Do not add another gateway. Use hosted checkout/tokenization and the fastest eligible rail: UPI intent, wallet, saved PayPal, or saved tokenized card. Card PAN/CVV never reaches the application server or logs.

## Production contract

1. The server calculates price, currency, shipping, tax, discount, and payable total from authoritative data. Never trust a client amount.
2. The server creates, captures, refunds, and verifies payments. Secrets remain server-side in environment-backed secret storage; test and live credentials never mix.
3. Every mutation has a persisted idempotency key. Retries, refresh, disconnect, and double-click reuse the same business operation.
4. Persist the gateway order/reference before client approval. On return or reload, resume by status; never create a replacement charge for an in-flight payment.
5. Fulfil only after a signature-verified webhook or server-to-server status verification confirms the expected merchant, order, amount, currency, and terminal state.
6. Process webhook events idempotently and tolerate duplicates, delays, and out-of-order delivery. A timeout is pending, not failed; reconcile until terminal.
7. Offers, EMI, refunds, settlement, and audit history are server-authoritative. Every state transition is traceable.
8. The UI exposes idle, processing, action-required, pending, success, recoverable failure, and terminal failure states; submit is guarded and accessible.
9. Preserve the project's database ownership and tenancy boundaries. For AiVibe, use the approved RDS PostgreSQL schema; DynamoDB is forbidden.

## Release gate

Verify sandbox scenarios first, then controlled live payment, webhook signature, duplicate event, repeated submit, refresh during approval, network loss, delayed webhook, amount mismatch, failed/partial refund, settlement reconciliation, international currency, and 3DS/SCA where applicable. Ship only with clean tests, logs, and reconciliation.
