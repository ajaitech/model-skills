# VibeMyCar Admin Panel

## Goal
Internal, role-gated back-office console for the VibeMyCar car-pooling platform. Used only by VibeMyCar's operations/support/finance staff (never riders or drivers) to manage users, ride/booking oversight, KYC review, vehicle catalog data, CMS/FAQ/notification content, payment-gateway configuration, and admin roles. A control-plane UI over the same Firestore project the mobile apps use, plus a small MySQL database for admin identity/RBAC only.

## Core requirements
- Every module route gated by the custom `permission:<module>,<route-key>` middleware tied to `roles`/`permissions` tables.
- Browser Firestore/Storage access authorized only via a Firebase custom token from `GET /firebase/token`, never a standing client credential.
- No live payment checkout may execute from this panel: legacy Paytm/Braintree(PayPal)/Stripe endpoints stay fail-closed (410/503); only the sibling Web Panel's authenticated payment API may charge.
- Payment secrets resolve only through `PaymentCredentialService` + `config/payment_boundary.php`; the browser sees only `configured: true/false`.
- Content edited here must stay shaped to the wire-contracts registry, since mobile clients read the same Firestore project.

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Language/runtime | PHP | `^8.1` | composer.json |
| Framework (incl. Blade templating, Sanctum tokens) | Laravel Framework / laravel/sanctum | `^10.0` / `^3.2` | composer.json |
| Authorization (declared, unused at runtime) | spatie/laravel-permission | `^5.11` | composer.json |
| Firebase server / client SDK | kreait/laravel-firebase / firebase JS (compat) | `^5.2` / unpinned | composer.json, firebase_client.blade.php |
| Google API client / HTTP client | google/apiclient / guzzlehttp/guzzle | `^2.15` / `^7.0.1` | composer.json |
| Payments (declared, checkout disabled here) | razorpay `2.*`, stripe-php `^15.0`, braintree_php `^6.7`, paypal-payouts-sdk `~1.0.0`, paytmchecksum `^1.1` | see cols | composer.json |
| Primary RDBMS | MySQL | `5.7.36` (reference dump), driver `mysql` | config/database.php, vibemycar_admin_database.sql |
| Frontend build / UI / CSS | laravel-mix / bootstrap / sass | `^6.0.6` / `^5.1.3` / `^1.32.11` | package.json |

## Architecture
Laravel 10 monolith as a pure **admin control plane**. MySQL holds only `users`/`roles`/`permissions` (custom schema — see gaps). Controllers only run `auth`/`permission` middleware and `return view(...)`; each Blade view embeds the Firebase Web SDK and reads/writes Firestore **directly from the browser** — Laravel never proxies app data. Bridge: browser calls `GET /firebase/token`, gets a Firebase custom token with `{admin:true, role:'admin'}` claims, then `firebase.auth().signInWithCustomToken()` (layouts/app.blade.php); field-level authorization is Firestore Security Rules, outside this repo. Payment secrets are the one server-kept exception: `config/payment_boundary.php` loads a JSON registry from `../../Firebase Import Export Collections/payment_boundary.json` (shared with the sibling panel), mapping provider secret fields to `VMC_PAYMENT_*` vars, exposed to Blade only as a `configured` boolean via `PaymentCredentialService`. Deployment **unverified** — no Dockerfile/CI; `.htaccess` implies conventional Apache/PHP-FPM hosting.

## Naming conventions
- Controllers: PascalCase + `Controller`, e.g. `VehicleBrandController.php`; Blade views: snake_case folders, e.g. `resources/views/vehicle_brand/index.blade.php`.
- Routes: kebab-case, dot.named, e.g. `->name('vehicle-model.edit')`; permission pair `<module>,<route-key>` e.g. `permission:vehicle-brand,vehicle.brand.edit`.
- Env vars: `VMC_PAYMENT_<PROVIDER>_<FIELD>` e.g. `VMC_PAYMENT_RAZORPAY_SECRET_KEY`; Firebase vars `FIREBASE_*`.
- Firestore collections: mixed snake_case/lowerCamel from the original template, e.g. `driver_users`, `bookedUser`, `vehicle_brand`.
- Wire-contract field paths: dot.path lowerCamel, e.g. `journeyMotion.reducedMotionPolicy`, `seatLock.ttlSeconds`.

## Data types & models
| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| Admin User | id, name, email, email_verified_at, password, role_id:int, remember_token | MySQL `users` | database/migrations/2014_10_12_000000_create_users_table.php |
| Role | id, role_name:string | MySQL `roles` | app/Models/Role.php |
| Permission | id, role_id:int, permission:string, routes:string | MySQL `permissions` | app/Models/Permission.php |
| Payment boundary | provider, documentId, checkoutAdapter, checkoutEnabledHere:bool, disabledCode, fields:{configured:bool} | computed (env + JSON registry) | app/Services/PaymentCredentialService.php |
| Settings docs (5: experienceConfig, providerConfig, notificationCommandConfig, releaseFeatures, vehicleExperience) | journeyMotion.*, providerConfig.auth/maps.*, releaseFeatures.features.{booking,wallet,sos,evCharging}, seatLock.{ttlSeconds,maxSeatsPerBooking} | Firestore `settings/<docId>` | resources/contracts/vibemycar_wire_contracts.json |
| Vehicle catalogs (3: vehicle_catalog, vehicle_seat_layout, vehicle_asset_3d) | vehicleCatalogId, make, model, modelYear:int, seatLayoutId, asset3DId; seats[]:{seatId,row,column,isDriver}; glbUrl, sha256, byteLength:int | Firestore, 3 collections | same |

## API surface
| Operation | Trigger / Path | Request | Response | Auth | Defined in |
|---|---|---|---|---|---|
| Mint Firebase custom token | `GET /firebase/token` | none | `{token, uid}` | `auth` | FirebaseTokenController.php |
| App data read/write + asset upload | Browser Firestore/Storage SDK, 25 collections incl. `users`, `driver_users`, `booking`, `documents`, `settings`, `sos`, `vehicle_brand` (full list: grep `.collection(`) | doc/binary | Firestore doc / download URL | Firebase custom-token session + Firestore Rules (not in repo) | resources/views/**/*.blade.php |
| Legacy checkout / payout (both fail-closed) | `POST payments/*`, `pay-to-user`, `check-payout-status` | any | 410/503 `{success:false,code}` | `auth`+`throttle:10,1` | PaymentController.php, UserController.php |
| Notify/email/delete-user | `POST send-notification`, `send-email`, `delete-user/{id}` (last calls FirebaseService::deleteUser) | form fields | redirect/JSON | no `permission:` wrapper | Notification/SendEmail/DeleteUserAuthenticationController.php |
| CRUD: users/roles/vehicle catalog/tax/currency/cms/faq/on-board/complaints/sos (~18 modules) | module routes | form/route params | Blade view | `permission:<module>,<route-key>` | routes/web.php |
| Sanctum user | `GET /api/user` | Bearer token | user JSON | `auth:sanctum` | routes/api.php |

## CORS & headers
`config/cors.php`: `paths=['api/*','sanctum/csrf-cookie']`, `allowed_methods=['*']`, `allowed_origins=['*']`, `allowed_headers=['*']`, `supports_credentials=false`, `max_age=0` — wildcard origin/headers, see gaps.

## Security boundary
- **Admin identity**: Laravel session guard on MySQL `users`; role/permission via the app's own `roles`/`permissions` tables (`role_id`,`permission`,`routes`) — custom, **not** the declared spatie/laravel-permission package.
- **Route authorization**: `permission:<module>,<route-key>` middleware on nearly every module route; unauthenticated → login, unauthorized → HTTP 403.
- **Firestore/Storage authorization**: short-lived Firebase custom token from `GET /firebase/token` carrying `{admin:true, role:'admin'}`; enforcement is Firestore Security Rules, outside this repo.
- **Secret sources (names only)**: 18 `VMC_PAYMENT_<PROVIDER>_*` vars, `FIREBASE_*` (9 vars), `AWS_*` (unused, no aws-sdk dep), `MAILGUN_*`, `POSTMARK_*`, `GOOGLE_MAP_KEY`, `APP_KEY`. `server_payment_credentials` Firestore collection is private per `payment_boundary.php`; browser only gets `configured` booleans.
- **Public vs private**: everything under `/users`, `/rides`, `/settings/*` requires an authenticated session; three POST routes lack a `permission:` wrapper (see API surface).

## Known gaps & risks
- Hardcoded FCM legacy server-key literal in `config/constant.php` (not sourced from `env()`) — real secret in source; rotate and move to env.
- CORS `allowed_origins`/`allowed_headers` wildcard (`['*']`) on `api/*` and `sanctum/csrf-cookie` (config/cors.php).
- `ApiKeyAuth.php`/`ApiSecureKeyAuth.php` reference undefined classes `BaseApiController`/`UserAccessToken` — dead code, unregistered in `Kernel.php`; also compares client `apikey` header to `env('APP_KEY')`, Laravel's encryption key.
- composer.json declares `spatie/laravel-permission ^5.11` and ships its migration, but production and all runtime code use an incompatible custom `roles`/`permissions` schema.
- No Dockerfile/CI config (deployment unverified); MySQL dump dated Aug 2022 / MySQL 5.7.36, drift vs. current migrations plausible and unverified.
- All primary app data (every module's Firestore reads/writes) happens directly from browser JS, no Laravel-side validation or audit trail.
