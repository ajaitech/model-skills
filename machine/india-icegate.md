# India ICEGATE — Customs EDI

Scope: purpose, filing categories, authentication model, and official portal root only. No invented endpoint paths or field-level API detail — ICEGATE's API/EDI surface requires registered-user access; verify every concrete detail against the current ICEGATE e-filing/API documentation before coding against it.

## Official portal root
| Purpose | Root |
|---|---|
| ICEGATE main portal (e-filing, tracking, registration) | https://www.icegate.gov.in |
Do not guess sub-paths, message formats, or API endpoints beyond this root from memory.

## What ICEGATE is
Indian Customs' Electronic Data Interchange (EDI) gateway — the interface between trade (importers, exporters, customs brokers, shipping lines, airlines) and the Indian Customs (CBIC) backend system (ICES). It is the mandatory electronic channel for customs clearance documentation; there is no meaningful paper-only alternative for commercial import/export.

## Filing categories (model)
| Category | Who files | Purpose |
|---|---|---|
| Bill of Entry (BoE) | Importer / Customs Broker | Declares imported goods for customs clearance and assessment |
| Shipping Bill | Exporter / Customs Broker | Declares goods being exported; needed for clearance and export-benefit claims |
| Import General Manifest (IGM) | Carrier (shipping line/airline) | Manifest of cargo arriving into India |
| Export General Manifest (EGM) | Carrier | Manifest of cargo leaving India, confirms export |
| Amendments, duty-payment challans, and other customs-adjacent messages | Multiple parties | The full EDI message-type taxonomy has evolved over time — verify the current, complete list against ICEGATE's own filing-category documentation rather than treating the rows above as exhaustive |

## Authentication model
- Access requires ICEGATE registration tied to an IEC (Importer-Exporter Code) or a Customs Broker license — this is a permissioned trade-community system, not an open public signup.
- Registered users get credential-based portal login; electronic filing for specific legally-binding document types has historically layered a Digital Signature Certificate (DSC) requirement on top of portal access.
- Programmatic/API access (as distinct from portal-form filing) is a separate onboarding track from ICEGATE/CBIC with its own registration and authentication contract. Exact current mechanism (API key, DSC-signed request, or other) must be verified against ICEGATE's current API documentation — do not assume portal login credentials are reusable for API access without confirming.

## Hard rule for this task
Treat any request for an exact ICEGATE endpoint URL, request/response JSON shape, or DSC-signing implementation detail as unverifiable from memory. Point to https://www.icegate.gov.in as the source of the current, authoritative spec — or, more practically, to the customs broker's existing EDI vendor integration docs, since most real-world integrations go through an established customs-broker software vendor rather than a fresh direct build.
