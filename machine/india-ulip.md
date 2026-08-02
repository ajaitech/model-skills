# India ULIP — Unified Logistics Interface Platform

Scope: purpose, API access model, and official portal root only. No invented endpoint paths or field-level API detail — verify every concrete detail against the current ULIP/NLDSL API documentation before coding against it.

## Official portal root
| Purpose | Root |
|---|---|
| ULIP portal | https://ulip.dpiit.gov.in — verify this is the current live host before use. ULIP sits under the Ministry of Commerce/DPIIT and National Logistics Policy umbrella, and portal/API/docs subdomains have shifted; confirm before hardcoding. |
Do not guess sub-paths or exact API endpoints beyond this root from memory.

## What ULIP is
A government logistics-data platform (under India's National Logistics Policy / PM Gati Shakti) exposing a single API layer that aggregates logistics and tracking data from multiple source systems — customs, ports, railways, road transport, and others — so a logistics or supply-chain application can query shipment status across modes without integrating each source system separately. It is a data-aggregation and interoperability layer, not a shipment-booking system (contrast with a carrier-specific booking API such as Blue Dart's).

## API access model
- Access is via **NLDSL** (National Logistics Data Services Limited, the entity operationalizing ULIP) onboarding — a registered-partner model, not open self-serve signup. An organization applies, is vetted, and is provisioned with credentials scoped to the data domains it's approved for.
- Because ULIP aggregates data contributed by multiple government/quasi-government source systems, the fields available for a given shipment depend on which source systems have reported into ULIP for that specific movement — coverage is not uniform across all transport modes or regions. Verify current coverage claims rather than assuming full nationwide parity.
- Auth mechanism (API key, OAuth-style token, or other) and rate limits are not fixed here — they are a partner-issued contract delivered at approval time. Verify against the current NLDSL onboarding/API documentation, not a generic public spec.

## Hard rule for this task
Treat any request for an exact ULIP endpoint URL, request/response schema, or auth-flow detail as unverifiable from memory. Direct the user to initiate NLDSL onboarding at the ULIP portal (root above) to obtain the current, authoritative integration spec — never fabricate a plausible-looking REST contract.
