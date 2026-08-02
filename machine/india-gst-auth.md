# India GST Authentication Model

Scope: the GST authentication + filing MODEL only. No invented endpoint paths, field names, or API shapes below — verify every concrete detail against the current GSP/API spec before coding against it.

## Official portal roots
| Purpose | Root |
|---|---|
| GST Common Portal (returns, registration, payments) | https://www.gst.gov.in |
| GSTN developer/API ecosystem info | https://www.gstn.org.in |
| E-Invoice system | https://einvoice1.gst.gov.in — verify current active host; GSTN/NIC has migrated e-invoice infra across multiple IRP hosts, do not hardcode without checking the current advisory |
| E-Way Bill system | https://ewaybillgst.gov.in |
Do not guess a sub-path (`/api/v1/...`) on any of these roots from memory — confirm against the current GSP/ASP integration spec for the specific flow being built.

## Actors in the model
- **Taxpayer**: the GSTIN-holding entity whose returns/invoices are filed.
- **GSP (GST Suvidha Provider)**: GSTN-authorized entity with direct API access to the GST system backend.
- **ASP (Application Service Provider)**: builds taxpayer-facing software on top of a GSP's APIs. Most third-party integrations, including typical SaaS, go through an ASP-GSP pair, not directly to GSTN.
- A production integration almost always means: your app (ASP layer) → contracted GSP → GSTN backend. Direct-to-GSTN access without a GSP is not the normal path for a non-GSP entity — verify current eligibility if considering it.

## Auth/session lifecycle — model, not exact fields
1. App authenticates to its contracted GSP with GSP-issued credentials. Exact grant type is GSP-specific — each GSP publishes its own auth contract on top of GSTN's; verify with that GSP's own spec.
2. Taxpayer-initiated portal login typically layers an OTP step (mobile/email registered against the GSTIN) on top of username/password. Exact OTP endpoint is GSP- and portal-version-specific — do not hardcode.
3. A successful auth yields a session/auth token with limited validity; refresh behavior is GSP-specific — verify against that GSP's docs, not GSTN's generic documentation.

## E-Invoice (IRN) flow — model only
- Applicable above the notified turnover threshold — this threshold has changed multiple times; verify the current figure before assuming a business is/isn't covered.
- Shape: generate invoice JSON in the currently notified schema version → submit to the IRP via GSP → receive signed IRN + QR code → IRN becomes mandatory on the tax invoice.
- Schema version changes periodically — verify current version against the GSP's or NIC's current schema doc, never reuse a cached schema from memory.

## E-Way Bill flow — model only
- Required for movement of goods above the notified value threshold; threshold and intra-state variations differ by state — verify the current state-specific rule, not just the central one.
- Shape: generate EWB request referencing the invoice/IRN → submit via GSP → receive EWB number + validity window tied to distance/transport mode.
- Validity-per-distance slabs have changed historically — verify the current slab before calculating an expected validity window.

## Hard rule for this task
Every field name, request/response shape, endpoint path, and OTP/token TTL above is explicitly unspecified — a real GSP contract is required for actual values. Treat any such detail a user requests as "verify against the current GSP spec," never fill it from memory.
