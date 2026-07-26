---
name: bluedart-shipping
description: >-
  Use when integrating or reviewing shipping/logistics for India via Blue Dart (Bluedart Express) —
  live shipping price/rate, pincode serviceability & transit time, waybill/AWB generation & live
  booking, pickup registration, tracking, cancellation, COD vs prepaid, and AWB label generation.
  The complete Bluedart API tech stack for accurate, uniform, instant ship-anywhere-in-India flows.
  Use with payment-gateway when shipping feeds checkout, and with world-class-design and
  ui-flow-review-loop for address and checkout UI. Exact endpoints, auth, and fields are in
  references/bluedart-api.md.
---

# Bluedart Shipping — live pricing, booking & tracking across India

## Overview

The goal: from an app, quote a **live shipping price**, confirm **serviceability + transit time** for the
destination pincode, **book a shipment (generate the waybill/AWB) live**, register **pickup**, **track**
to delivery, and **cancel** when needed — instantly, anywhere Blue Dart serves in India, with **uniform**
integration and **accurate** data. Full auth flow, base URLs, endpoints, request/response fields, and
service/product codes are in [`references/bluedart-api.md`](references/bluedart-api.md).

**Flow (the order of operations):**
1. **Serviceability + transit time** for the destination pincode (fail early if not serviceable).
2. **Live price** — computed by a **server-side rate-card** (product + zone [origin→dest pincode] +
   chargeable weight + COD surcharge). Blue Dart exposes **NO rate API** — see the reference; do not hunt
   for one. Gated by the serviceability + transit result.
3. Feed that cost into the order total → **`payment-gateway`** (pay).
4. **Waybill/AWB generation & booking** (idempotent — one AWB per order).
5. **Pickup registration**.
6. **Tracking** to delivery; **cancellation** if required.

## STRICT RULES (non-negotiable) (BD#)
1. **Serviceability pre-check.** Never quote or book without first confirming the destination pincode is
   serviceable and getting the transit time. Surface non-serviceable clearly. (BD1)
2. **Idempotent booking — one AWB per order.** Guard waybill generation so a retry/refresh/double-click
   NEVER creates a duplicate AWB/shipment. Persist the AWB against the order; if a booking call times out,
   reconcile (query) before retrying — the shipment may already exist. (BD2)
3. **Server-authoritative price.** Blue Dart has **no rate API** — compute the shipping price **server-side
   from your negotiated rate-card** (product + zone + chargeable weight = max(actual, volumetric)); never
   trust a client-sent shipping amount feeding the order total. (BD3)
4. **Address & data accuracy.** Validate pincode ↔ city/state, phone, and mandatory fields before booking;
   accurate weight + dimensions (volumetric) or the rate/booking is wrong. (BD4)
5. **Credentials are secrets.** LicenseKey / LoginID / ClientID / secret come from env / Secrets Manager —
   NEVER hardcoded. Cache the JWT token and refresh before expiry; never log it. (BD5)
6. **Sandbox vs production hygiene.** Separate sandbox and live credentials + base URLs; never book live
   from a test path. (BD6)
7. **Resilient calls.** Retry transient failures with backoff (but idempotently for booking); handle and
   surface Blue Dart error codes; reconcile status via tracking rather than assuming. (BD7)
8. **Uniformity.** One typed client wraps every Blue Dart call (auth, rate, serviceability, booking,
   pickup, track, cancel) — no scattered ad-hoc requests; belongs in the API layer's external branch
   (`external/l2` non-government, per `elite-launch` layered architecture). (BD8)

## Self-check — before shipping code ships
Serviceability + transit checked before quote/book · rate confirmed server-side · booking idempotent (one
AWB/order, reconcile before retry) · address/weight/dimensions validated · credentials from Secrets Manager,
JWT cached + refreshed, never logged · sandbox/live separated · one typed client in the external API branch ·
tracking + cancellation wired · shipping cost flows correctly into `payment-gateway` order total.
