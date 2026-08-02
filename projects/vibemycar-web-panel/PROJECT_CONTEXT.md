# VibeMyCar Web Panel

## Goal
A Laravel 12 customer-facing web app for the VibeMyCar car-pooling platform.
Riders/drivers sign in via Firebase Auth (email/password, or social/phone via
`POST /firebase.login`), find/publish shared rides, manage bookings, top up
an in-app wallet, and pay via a server-authoritative Razorpay/Stripe checkout
API. Public rider/driver surface, not an admin tool. Firestore (project
`vibemycar`) is the system of record for rides, bookings, wallets and
payments; local MySQL holds only Laravel's own `users`/`sessions`/`cache`/
`jobs` tables, bridging Firebase identity to a session.

## Core requirements
- Payment settlement is server-authoritative: verify the provider payment
  against live Firestore booking state in a transaction before crediting any
  wallet (`FirestorePaymentCheckoutStore::settle()`).
- Only Razorpay and Stripe confirm payment server-side; every other gateway
  returns `SERVER_CONFIRMATION_UNAVAILABLE` (`checkoutAdapters` in
  `payment_boundary.json`).
- Every `/v1` call needs a Firebase ID token (Bearer) **and** an App Check
  token (`X-Firebase-AppCheck`), or 401 (`FirebaseAuthenticate`).
- Checkout/settlement is idempotent via `clientRequestId`/`fingerprint` plus
  a Firestore transaction lease. Legacy browser payment routes stay retired
  (HTTP 410); session pages redirect unauthenticated visitors to `login`.

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Language | PHP | `^8.2` | `composer.json:29` |
| Framework | Laravel | `v12.55.1` | `composer.lock` |
| Firebase Admin | `kreait/laravel-firebase`/`firebase-php` | `6.2.0`/`7.24.1` | `composer.lock` |
| Firestore client | `google/cloud-firestore` | `v1.55.0` | `composer.lock` |
| Payments (live) | `razorpay/razorpay`, `stripe/stripe-php` | `2.9.2`, `v17.6.0` | `composer.lock` |
| Frontend build | Vite / `laravel-vite-plugin` | `6.3.5` / `1.2.0` | `package-lock.json` |
| CSS | Tailwind + `@tailwindcss/forms` | `3.4.17` + `0.5.10` | `package-lock.json` |
| Interactivity | Alpine.js + `@alpinejs/focus` | `3.14.9` | `resources/js/app.js:2` |
| Client Firebase SDK | Firebase JS (compat), CDN, not npm | `9.22.1` | `layouts/app.blade.php:353-356` |
| Local store | MySQL `vibemycar_web` | driver `mysql` | `.env`; `config/database.php:20` |

## Architecture
Laravel MVC, Blade + Alpine.js. `bootstrap/app.php` wires `routes/web.php`
(session pages, `RedirectIfNotAuthenticated`) and `routes/api.php` (stateless
`/v1`, `firebase.auth` -> `FirebaseAuthenticate`). Browser signs in with the
CDN Firebase JS SDK, then either posts the ID token to `POST /firebase.login`
(starts a session) or calls `/v1` with ID token + App Check token (no
session). Business data lives in Firestore (project `vibemycar`) via the
Admin SDK, or (one public path) raw REST
(`app/Helpers/FirestoreHelper.php`). Payments are hexagonal:
`Contracts\Payments\*` (ports) -> `Domain\Payments\*` (value objects) ->
`Services\Payments\*` (`PaymentGatewayRegistry`, gateways) ->
`Infrastructure\Payments\*` (`FirestorePaymentCheckoutStore`, SDK clients).
`config/payment_boundary.php` loads its credential registry from **outside
this repo** (`../../Firebase Import Export Collections/payment_boundary.json`),
shared with the mobile app and Admin Panel. Deploys to Apache/mod_rewrite.

## Naming conventions
- Controllers: `{Noun}Controller` (`RideController`, `PaymentCheckoutController`).
  Domain objects: `final readonly class` or backed enum; provider slugs are
  idiosyncratic — `PaymentProvider::Stripe = 'strip'` (`PaymentProvider.php:9`),
  matching Firestore field key `"strip"`.
- Firestore collections: `snake_case` multi-word (`wallet_transaction`),
  single word otherwise (`booking`, `users`); document fields `camelCase`
  (`amountMinor`, `walletAmount`). API responses share one envelope:
  `{accepted, status, settled, session|providers|settlement, error}`.
- Payment secrets: `VMC_PAYMENT_{PROVIDER}_{FIELD}`; `PaymentCredentialService`
  refuses names not starting `VMC_PAYMENT_`.

## Data types & models
| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| `User` (local) | id, name, email, password(hashed), firebase_uid, phone_number, login_provider | MySQL `users` | `app/Models/User.php` |
| `CheckoutSession` | id, uid, clientRequestId, provider, purpose, amountMinor:int, currency, status, bookingId?, clientPayload | Firestore `payment_checkout_session` | `CheckoutSession.php` |
| `InternalBookingSettlement` | id, uid, bookingId, method:wallet\|cash, amountMinor:int | same collection (kind=internalBookingSettlement) | `InternalBookingSettlement.php` |
| `bookedUser` (external) | paymentStatus:bool, subTotal, taxList:array, adminCommission | Firestore `booking/{id}/bookedUser/{uid}` | `BookingPricing::totalMinor()` |
| `users`/`wallet_transaction` (external) | walletAmount:decimal / amount, userId, type, isCredit:bool | Firestore `users`, `wallet_transaction` | `FirestorePaymentCheckoutStore` |

## API surface
`env` = envelope `{accepted,status,settled,session|providers|settlement,error}`.
`auth+AC` = `firebase.auth` middleware + App Check token.

| Operation | Trigger/Path | Request shape | Response | Auth | Defined in |
|---|---|---|---|---|---|
| List/create/get/confirm checkout | `/v1/payment/providers`, `/checkout-sessions[/{id}[/confirm]]` | create `{provider,purpose,clientRequestId,amountMinor?,currency?,bookingId?}`; confirm `{paymentId,orderId,signature}`/`{}` | `env` | auth+AC | `PaymentCheckoutController` |
| Settle wallet/cash | `POST /v1/payment/internal-booking-settlements` | `{bookingId,method,clientRequestId}` | `env{settlement}` | auth+AC | `::settleInternalBooking` |
| Trigger notification | `POST /v1/notification-commands` | `{command,bookingId,recipientId,clientCommandId}` | `{accepted,commandId,status}` | auth+AC | `NotificationCommandController` |
| Firebase session login | `POST /firebase.login` | `{id_token,loginType,name?,email?}` | `{success,redirect}` | public | `LoginController::firebaseLogin` |
| Legacy wallet/order routes (13) | `wallet-topup-*`, `orderpayment-*` | any | HTTP 410 `env` | session | retired stubs |
| Public Firestore doc read (no route) | `FirestoreHelper::getPublicDocument(path)` | path | field map or null | none — raw REST | `app/Helpers/FirestoreHelper.php` |

6 rows, 9 ops (a 10th, `POST /sendnotification`, is under Known gaps);
`::confirm` also calls the Razorpay/Stripe SDK server-to-server.

## CORS & headers
No `config/cors.php` (only the unpublished framework default);
`bootstrap/app.php` registers no `HandleCors` middleware or origin policy —
**none configured — GAP**. `Authorization: Bearer` / `X-Firebase-AppCheck`
are enforced by app code, not CORS. `public/.htaccess` forwards those
headers to PHP under Apache.

## Security boundary
- `/v1/*`: verified Firebase ID token (`FirebaseAuthService::verifyIdToken`)
  **and** valid App Check token; UID taken only from the token's `sub` claim.
- `/firebase.login` (weaker — see gaps): does **not** verify the token;
  trusts the raw client `id_token` as the `users.firebase_uid` key
  (`LoginController.php:37,48`).
- Session pages: Laravel session guard (MySQL) + `RedirectIfNotAuthenticated`;
  Breeze's email+password login runs in parallel (`routes/auth.php`).
- No role/permission package here (contrast Admin Panel's
  `spatie/laravel-permission`) — every authenticated user has equal access.
- Secrets: env-var only, via `PaymentCredentialService` (rejects names not
  under `VMC_PAYMENT_*`); names seen, no values: `FIREBASE_CREDENTIALS`,
  `AWS_ACCESS_KEY_ID/SECRET`, `GOOGLE_MAP_KEY`. Public: `landing`,
  `page/{slug}`, `privacy`, `terms`; everything else requires auth.

## Known gaps & risks
- **`/firebase.login` never verifies the ID token** — stores the raw
  client-supplied token as `users.firebase_uid` instead of a verified `sub`
  claim (unlike `/v1`); tokens rotate, so this likely mints duplicate users.
  (`app/Providers/FirebaseUserProvider.php` is an unreferenced, empty
  0-byte file.)
- **`resources/js/app.js` imports the npm `firebase` package**, absent from
  `package.json`/lock (Firebase is CDN-loaded instead) — unresolvable by
  Vite; also reads Mix-style `process.env.MIX_FIREBASE_*`, never populated
  under Vite — dead/broken code.
- **Mock payment code ships in `layouts/app.blade.php`**: a simulated
  FlutterWave success flow (`flw_ref: 'MOCK_SUCCESS_'+Date.now()`), though
  that gateway is server-disabled.
- **TLS verification disabled on an outbound call**:
  `NotificationAndMailController::sendnotification` sets
  `CURLOPT_SSL_VERIFYHOST=0`/`VERIFYPEER=false` calling `fcm.googleapis.com`
  — MITM exposure, and a weaker second push path beside
  `NotificationCommandController`.
- Local `.env` (git-ignored, untracked) has a populated `GOOGLE_MAP_KEY` —
  verify it's restricted. Web supports **both Razorpay and Stripe live**,
  unlike the Razorpay-only Flutter app — confirm intended.
