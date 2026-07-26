# Blue Dart Express (India) API — tech stack (2026, sourced)

Official docs live on the **DHL Group API Developer Portal** (`developer.dhl.com`, "DHL eCommerce India /
Blue Dart" family) — detailed spec pages are **login-gated**. Structure below is assembled from the public
overview pages + a working open-source JWT client + integrator onboarding docs. **Confidence flags:**
`HIGH` (≥2 sources incl. official) · `MED` (single credible / client only) · `UNVERIFIED` (gated/inferred).
Verify exact paths/field casing against your account's logged-in Swagger before go-live.

Build on the **modern REST/JSON gateway (APIGEE), JWT-authenticated** — not the legacy SOAP/WSDL.

## Base URLs (HIGH)
| Env | `{base}` |
|---|---|
| Sandbox | `https://apigateway-sandbox.bluedart.com/in/transportation` |
| Production | `https://apigateway.bluedart.com/in/transportation` |

## Auth — two credential layers (HIGH)
1. **Gateway OAuth** — register a DHL-portal app → `ClientID` (consumer key) + `clientSecret` → mint a **JWT**.
2. **Account profile** — every business **payload** also carries a `Profile` with `LoginID` + `LicenceKey`.

**Token (MED — path from client; official page gated):**
```
GET {base}/token/v1/login
  ClientID: <consumer key>      clientSecret: <consumer secret>   Accept: application/json
→ { "JWTToken": "<jwt>" }
```
**Business calls:** header `JWTToken: <jwt>` + `Content-Type: application/json`.
- **Token TTL is UNVERIFIED** (gated). Do NOT hardcode a lifetime — **decode the JWT `exp` claim and refresh
  proactively** (client refreshes within 30s of exp), cache in memory, **retry once on 401/403** after
  re-auth. Never log the token. (HIGH)
- **Profile object (in the BODY, not header)** (HIGH): `{ "LoginID": "...", "LicenceKey": "...", "Api_type": "S" }`
  — note British spelling **`LicenceKey`**; `Api_type` = `"S"` sandbox / production value as issued.
- **Gotcha — profile wrapper casing varies per endpoint** (HIGH): Waybill & Pickup use capital **`Profile`**
  inside `{ "Request": {...}, "Profile": {...} }`; Transit-Time / Location-Finder / Master-Download use
  lower-case **`profile`** at the payload root (no `Request` wrapper). Match each endpoint exactly.

## Endpoints (paths relative to `{base}`)
| Flow | Method | Path | Conf |
|---|---|---|---|
| Token | GET | `/token/v1/login` | MED |
| Serviceability (Location Finder) | POST | `/finder/v1/GetServicesforPincode` · `/GetServicesforProduct` · `/GetServicesforPincodeAndProduct` | MED-HIGH |
| Transit time | POST | `/transit/v1/GetDomesticTransitTimeForPinCodeandProduct` | MED |
| **Rate / pricing** | — | **NO PUBLIC API — use a server-side rate-card (see below)** | HIGH |
| Waybill / AWB generate | POST | `/waybill/v1/GenerateWayBill` | HIGH |
| Waybill cancel | POST | `/waybill/v1/CancelWaybill` | HIGH |
| Pickup register | POST | `/pickup/v1/RegisterPickup` | HIGH |
| Pickup cancel | POST | `/cancel-pickup/v1/CancelPickup` | MED |
| Tracking | GET | `/tracking/v1?...` | HIGH |
| Pincode master | POST | `/masterdownload/v1/DownloadPinCodeMaster` | HIGH |
| Product/Sub-Product master | POST | (Product-and-Sub-Product-Pickup-Detail API — valid codes at runtime) | MED |

> Path-spelling to confirm at integration: transit shows as `/transit/v1/...` in the client but `time-finder/
> v1` on a portal summary; serviceability has both `finder/v1` and `masterdownload/v1`. Verify on your live portal.

### Serviceability (POST `/finder/v1/GetServicesforPincode`)
Request: `pinCode` (6-digit), optional `pProductCode`/`pSubProductCode`/`PackType`/`Feature`, `profile` (incl.
`Version`). Response: `PinCode`, `PincodeDescription`, `AreaCode`, `ServiceCenterCode`, per-service flags
(`DomesticPriorityInbound/Outbound`, `ApexInbound`, `GroundInbound/Outbound`, `eTailCODInbound`…). Capture the
returned **`AreaCode`** — you need it for booking/pickup. (fields MED)

### Transit time (POST `/transit/v1/GetDomesticTransitTimeForPinCodeandProduct`)
Request: `pPinCodeFrom`, `pPinCodeTo`, `pProductCode`, `pSubProductCode`, `pPudate` (Blue Dart date
`"/Date(<ms>)/"`), `pPickupTime` (`"HH:MM"`), `profile`. Response: expected delivery `DateTime`. (HIGH)

### Waybill / AWB generate & book (POST `/waybill/v1/GenerateWayBill`)
Envelope `{ "Request": { Shipper, Consignee, Returnadds, Services }, "Profile": {…} }`. Key fields (HIGH):
- **Shipper:** `OriginArea` (3-letter, e.g. `BOM`/`DEL` — must match shipper pincode region), `CustomerCode`
  (Prepaid or COD customer code), `CustomerName`, `CustomerAddress1..3`, `CustomerPincode`, `CustomerMobile`,
  `IsToPayCustomer`(bool), `CustomerGSTNumber?`.
- **Consignee:** `ConsigneeName`(≤30), `ConsigneeAddress1..3`(≤90), `ConsigneePincode`, `ConsigneeMobile`,
  `ConsigneeAddressType`(`'R'`/`'C'`), `ConsigneeGSTNumber?`.
- **Services:** `ProductCode`, `ProductType`(`0`=DOX docs / `1`=NDOX parcels), `SubProductCode`(`'P'`prepaid/
  `'C'`COD), `PieceCount`, `ActualWeight`(**kg**), `DeclaredValue?`, `CollectableAmount`(COD amount; `0`
  prepaid), **`CreditReferenceNo`** (your **unique per-shipment** ref, ≤20 — the idempotency key),
  `Dimensions[]`(`Length`/`Breadth`/`Height` in **cm** + `Count`), `PickupDate`, `PickupTime`,
  `RegisterPickup`(bool — book pickup in the same call), `PDFOutputNotRequired`(bool), `itemdtl[]`(e-waybill/GST).
- **Response** (MED): `AWBNo`, a token number, **`AWBPrintContent`** = label as **base64 PDF** (large; suppress
  with `PDFOutputNotRequired:true`). Errors in `Status[].StatusInformation` / `IsError`.

### Pickup register (POST `/pickup/v1/RegisterPickup`)
Request (HIGH): `ProductCode`, `AreaCode`(matches pickup pincode), `CustomerCode`, `CustomerName`,
`CustomerAddress1..2`, `ContactPersonName`, `CustomerPincode`, `MobileTelNo`, `ShipmentPickupDate`
(`YYYY-MM-DD`), `ShipmentPickupTime`(`HH:MM`), `NumberofPieces`, `WeightofShipment`(kg), `VolumeWeight`
(volumetric kg), `RouteCode?`(default `"99"`), `Profile`. Response: pickup **Token Number** + Status.

### Tracking (GET `/tracking/v1`) — GET only, query params + JWTToken header (HIGH)
```
GET {base}/tracking/v1?handler=tnt&action=custawbquery&loginid=<LoginID>&lickey=<LicenceKey>
    &awb=awb&numbers=<AWBNo>&verno=1&scan=1&format=json      (Header: JWTToken: <jwt>)
```
`scan=1` full scan events / `scan=0` latest only. Legacy no-JWT equivalent: `https://api.bluedart.com/servlet/
RoutingServlet?handler=tnt&action=custawbquery&loginid=<id>&awb=awb&numbers=<awb>&format=xml&lickey=<key>&verno=1.3`.

### Cancellation
- **AWB:** POST `/waybill/v1/CancelWaybill`, `{ "Request": { "AWBNo": "<awb>" }, "Profile": {…} }` (before in-scan). (HIGH)
- **Pickup:** POST `/cancel-pickup/v1/CancelPickup`, `{ "Request": { "TokenNumber": "<token>", "ShipmentPickupDate": "YYYY-MM-DD" }, "Profile": {…} }`. (MED)

## Product / sub-product codes, COD vs prepaid, weight
- **Product codes** (HIGH): `A` Air/Apex · `D` Domestic Priority · `E` Ground/Surface · `I` International.
  **Don't hardcode** — the Product/Sub-Product master API returns the authoritative valid combos for your
  account; drive booking from it.
- **SubProductCode** (MED): `P` prepaid · `C` COD. **Topay** = `IsToPayCustomer`/`isToPayShipper=true` (not a
  letter). COD needs a separate **COD Customer Code** + `CollectableAmount>0`; prepaid uses the Prepaid code +
  `CollectableAmount=0`. `ProductType`: `0`=DOX, `1`=NDOX.
- **Weight/volumetric** (HIGH): weight in **kg**, dims in **cm**. **Chargeable = max(actual, volumetric)**;
  volumetric = (L×B×H cm) ÷ **5000** per piece (confirm your contract divisor). Send accurate values or the
  price/booking is wrong.

## No public Rate API — live pricing = server-side rate-card (HIGH, load-bearing)
Blue Dart's API family has **no rate endpoint** (the bluedart.com "Price Finder" is a website tool, not an
API). Implement "live price" as a **server-side rate-card calculation** keyed on **product + zone (origin→dest
pincode) + chargeable weight + COD surcharge**, gated by a Location-Finder serviceability check and Transit-
Time. This keeps pricing **server-authoritative** (BD3) and correct. (Aggregators like Shiprocket/NimbusPost
wrap Blue Dart with their own rate APIs — only if you ship via an aggregator account.)

## Best practices
- **Serviceability pre-check first** (Location Finder / pincode master); capture `AreaCode`/`ServiceCenterCode`.
- **Idempotent booking**: `CreditReferenceNo` must be **unique per shipment** — reuse returns "Waybill already
  generated." Persist AWB↔order; on booking **timeout, reconcile via tracking/query before retrying**.
- **`OriginArea`/`AreaCode` must match the shipper/pickup pincode region** or you get `InvalidAreaScNotInRegion`.
- **Errors come inside HTTP 200** — check `IsError`/`Status[].StatusInformation`/`ErrorMessage`; 200 ≠ success.
- **Dates** serialise as `"/Date(<ms-since-epoch>)/"` — convert ISO before sending.
- **Label PDF** `AWBPrintContent` is base64 and large — set `PDFOutputNotRequired:true` when unneeded; render
  custom labels in your app layer.
- **Rate limits/quotas UNVERIFIED** — assume per-app APIGEE quotas; backoff. Sandbox vs prod: separate creds +
  base URLs, `Api_type="S"` in sandbox, never book live from a test path. **One typed client** wraps everything (BD8).

## Onboarding
- **Blue Dart account creds** (`LoginID`, `LicenceKey`, Prepaid + COD Customer Codes): request via your Blue
  Dart **sales rep / API support**; sandbox issued first → generate a sample label → submit request/response
  samples for validation → production creds issued.
- **DHL portal OAuth app** (`ClientID`/`clientSecret`): `developer.dhl.com` → My Apps → Add Developer App →
  select the Blue Dart / DHL eCommerce India APIs → get Consumer Key/Secret for the JWT.
- No official first-party REST SDK / public Postman found; portal has interactive "Try it" consoles when
  logged in. Community refs: `nidhiyashwanth/bluedart-mcp-server` (modern JWT/REST TS), `piyushdolar/bluedart-php-api` (legacy SOAP).

## Sources
developer.dhl.com/api-reference (waybill, transit-time, location-finder, registration-pickup, pickup-
cancellation, shipment-tracking, master-download, product-and-sub-product-detail, blue-dart-authentication —
several login-gated) · github.com/nidhiyashwanth/bluedart-mcp-server · github.com/piyushdolar/bluedart-php-api ·
support.aftership.com (Blue Dart credentials guide) · support.unicommerce.com (Blue Dart integration) ·
bluedart.com/price-finder (web tool).
