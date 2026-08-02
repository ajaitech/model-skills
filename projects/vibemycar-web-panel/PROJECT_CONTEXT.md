# VibeMyCar Web Panel

## Goal
Public rider/driver web app for VibeMyCar car-pooling, not an admin tool.
Users sign in with Firebase Auth; the **browser then talks to
Firestore directly** (project `vibemycar`, compat JS SDK) for rides, bookings,
chat and wallet. Laravel serves Blade shells, holds the session, and owns only
two server-authoritative concerns: the `/v1` payment-checkout and notification
APIs. MySQL holds only Laravel's own auth/session tables.

## Core invariants
- Firestore Security Rules are the real authorization boundary: Blade + JS
  read/write ~25 collections straight from the browser (`users`, `booking`,
  `chat`, `inbox`, `settings`, `currency`…) under rules outside this repo
  (`../../firestore.rules`). Page controllers return a view and nothing else.
- Settlement is server-authoritative: `/v1` re-reads the provider payment and
  live booking state in one Firestore transaction, rejecting on amount or
  currency drift (`FirestorePaymentCheckoutStore::settle()` :226-241).
  Idempotency key: `sha256("externalProviderCheckout\0uid\0clientRequestId")`,
  plus a `creationLease` UUID and a per-transaction `fingerprint` check.
- Only `razorpay` and `strip` (Stripe) have server adapters; the other 16
  registry entries return `SERVER_CONFIRMATION_UNAVAILABLE`. **Stripe is a
  sanctioned live gateway here**, unlike the Razorpay-only Flutter app.

## Tech stack
| Layer | Technology (locked in `composer.lock` / `package-lock.json`) | Version |
|---|---|---|
| Language | PHP (`composer.json:29`) | `^8.2`, deps cap 8.5 |
| Framework | Laravel | `v12.55.1` |
| Firebase Admin | `kreait/laravel-firebase` / `firebase-php` | `6.2.0` / `7.24.1` |
| Firestore (server) | `google/cloud-firestore` | `v1.55.0`, **needs `ext-grpc`** |
| Payments (live) | `razorpay/razorpay`, `stripe/stripe-php` | `2.9.2`, `v17.6.0` |
| Client Firebase | JS **compat** via CDN (`layouts/app.blade.php:353-356`) | `9.22.1` |
| Real frontend | vendored jQuery + Bootstrap (`public/assets/vendor/`); Vite/Tailwind/Alpine declared but never loaded — gap 4 | `3.7.1`+`5.3.3` |
| Local store | MySQL via `DB_CONNECTION` (`config/database.php:20`) | 5 migrations |

## Architecture
`bootstrap/app.php` wires `routes/web.php` (session pages) and `routes/api.php`
(stateless `/v1`, `throttle:30,1` + `firebase.auth`), pushes
`FirebaseMaintenance` + `SetLocale` onto the `web` group, exposes `/up`.
Payments are hexagonal: `Contracts\Payments\*` ports -> `Domain\Payments\*`
value objects -> `Services\Payments\*` gateways -> `Infrastructure\Payments\*`
(`FirestorePaymentCheckoutStore`, SDK clients), bound in `AppServiceProvider`.
`config/payment_boundary.php:5` reads its adapter/credential registry
(`schemaVersion` 2, 18 providers) from **outside this repo**:
`../../Firebase Import Export Collections/payment_boundary.json`. Apache/
mod_rewrite; `public/.htaccess` re-exports only `Authorization` and
`X-XSRF-Token`. Provider slugs are idiosyncratic: `PaymentProvider::Stripe =
'strip'` :9, matching the Firestore key.

## Build & run
| Step | Command | Note |
|---|---|---|
| PHP deps | `composer install` | **fails without `ext-grpc`**; add `ext-protobuf` |
| Env | `cp .env.example .env && php artisan key:generate` | committed `.env` has blank `DB_DATABASE`/`DB_USERNAME` |
| Schema | `php artisan migrate` | 5 migrations; `users.email` is `unique()` |
| Firebase | service account JSON at `storage/app/private/firebase/credentials.json` or `FIREBASE_CREDENTIALS` | gitignored |
| Run | `composer run dev` | `artisan serve` + `queue:listen` + `npm run dev` |
| Test/lint | `php artisan test` (sqlite `:memory:`) · `vendor/bin/pint` | |
| Assets | `npm run build` | **broken, unnecessary** — gap 4 |

## API surface
Every `/v1` route needs a verified Firebase ID token (Bearer) **and** an App
Check token (`X-Firebase-AppCheck`), else 401 (`FirebaseAuthenticate`).
Envelope: `{accepted,status,settled,session|providers|settlement,
error{code,message,retryable}}`; notifications `{accepted,commandId,status,code?}`.

| Path | Request |
|---|---|
| `GET /v1/payment/providers` | — |
| `POST /v1/payment/checkout-sessions` | `{provider,purpose:walletTopUp\|booking,clientRequestId:uuid,amountMinor?:int,currency?,bookingId?}`; wallet/booking fields mutually exclusive |
| `GET\|POST /v1/payment/checkout-sessions/{id:[a-f0-9]{64}}[/confirm]` | Razorpay `{paymentId,orderId,signature}`; Stripe `{}` (server re-reads the intent) |
| `POST /v1/payment/internal-booking-settlements` | `{bookingId,method:wallet\|cash,clientRequestId}` |
| `POST /v1/notification-commands` | `{command,bookingId,recipientId,clientCommandId}`, role-checked vs the booking |
| `POST /firebase.login` (**public**) | `{id_token,loginType,name?,email?,phoneNumber?,countryCode?}` — hazard 1 |
| `POST /sendnotification` (session) | unvalidated `{fcm,title,message,payload}` — hazard 2 |
| 22 `wallet-*` / `orderpayment-*` / `ride/*` (session) | any — HTTP **410** |

Entities. MySQL `users` adds `firebase_uid`, `phone_number`, `login_provider`
to the stock columns. Booking price is derived server-side from
`booking/{id}/bookedUser/{uid}` (subTotal + taxList + adminCommission); wallet
balance is `users.walletAmount`.

## Security boundary
- `/v1`: UID comes only from the token `sub` claim (the `firebase_uid` request
  attribute); no client-sent uid is ever trusted.
- Session pages: session guard + `RedirectIfNotAuthenticated`; Breeze
  email+password login also exists (`routes/auth.php`). No RBAC — every
  authenticated user has identical access. Public: `/`, `landing`,
  `page/{slug}`, `privacy`, `terms`, `lang/change`, `socialsignup`, Breeze
  guest routes, `/firebase.login`.
- No `config/cors.php` (unpublished by default) and no `HandleCors` —
  **no origin policy configured**.
- Env names only, never values: `FIREBASE_CREDENTIALS`, `FIREBASE_API_KEY`,
  `FIREBASE_PROJECT_ID`, `GOOGLE_MAP_KEY`, `GOOGLE_CLIENT_ID`,
  `VMC_PAYMENT_{PROVIDER}_{FIELD}` (`PaymentCredentialService:18` rejects other
  prefixes). `AWS_*`/SES entries are stock scaffolding, unused.

## Known gaps & hazards
1. **`/firebase.login` authenticates nobody.** `login.blade.php:334` sends
   `id_token: user.uid` — a bare UID, not a token — and
   `LoginController.php:37,48` uses that raw string as the `users.firebase_uid`
   lookup/insert key; `FirebaseAuthService` is imported, never called. POSTing
   any known UID (browser-readable: the login page itself queries Firestore
   `users`) yields `Auth::login()` as that person — impersonation of every
   session page, `/profile/*` and `/sendnotification`. Fix: verify the ID token,
   key on `sub` (`FirebaseAuthenticate:39`).
2. **TLS verification disabled** — `NotificationAndMailController:53-54` sets
   `CURLOPT_SSL_VERIFYHOST=0`/`VERIFYPEER=false` calling `fcm.googleapis.com`,
   and accepts any caller-supplied `fcm` token unchecked;
   `NotificationCommandController` is the safe equivalent.
3. **`php artisan config:cache` silently breaks the client app.**
   `layouts/app.blade.php:369-374` and `rides/show.blade.php:1485` call `env()`
   in Blade; with config cached `.env` is not loaded, so `firebaseConfig` and
   the Maps key render empty (laravel.com/docs/12.x/configuration).
4. **The Vite toolchain is dead code**: no `@vite` in any Blade, no
   `public/build`, no compiled Tailwind. `resources/js/app.js` imports
   `@alpinejs/focus` (:3) and `firebase/app|auth|database` (:39-41), absent from
   `package.json`, and reads Mix-era `MIX_FIREBASE_*` — `npm run build` fails.
5. **Mock payment code ships in production markup**: a simulated FlutterWave
   success flow in `layouts/app.blade.php:288-340` (`flw_ref:
   'MOCK_SUCCESS_'+Date.now()`; `window.flutterwaveIsSimulated` :1065). The
   layout also loads Paystack, MercadoPago and a **sandbox** Midtrans SDK
   (:349); all are server-disabled.
6. Operational traps: `config/payment_boundary.php` reads the sibling JSON with
   no existence check, so a missing sibling repo fatals every request;
   `activeCurrency()` needs **exactly one** Firestore `currency` doc with
   `enable == true`, else 503 on all checkout;
   `FirebaseMaintenance` blocks every web request on a Firestore REST GET of
   `settings/globalValue` (5 s, 2 retries).
7. `UserController::update:41-46` writes `first_name`/`last_name`, neither
   columns nor `$fillable` — silently dropped — and never writes to Firestore,
   while `index()` reads the profile *from* it, so edits do nothing.
