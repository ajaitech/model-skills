# VibeMyCar Admin Panel

## Goal
Internal, role-gated back-office console for VibeMyCar ops/support/finance staff only, never riders or drivers. A control plane over the Firestore project the mobile apps use, plus a small MySQL database for admin identity and RBAC.

## Core requirements
- Gate every module route with `permission:<module>,<route-key>` (75 groups, `routes/web.php`), and authorize browser Firestore/Storage only by the `GET /firebase/token` custom token — never a standing credential.
- Never re-enable checkout; only the sibling Web Panel's authenticated payment API may charge.
- Keep edited content shaped to the wire contracts; mobile clients read the same documents.

## Tech stack
Constraint/lock, from composer.json, composer.lock, package.json.
| Layer | Technology | Version |
|---|---|---|
| Language | PHP | `^8.1` |
| Framework + API tokens | laravel/framework, laravel/sanctum | `^10.0`/v10.13.0, `^3.2`/v3.2.5 |
| Authorization pkg — declared, unused at runtime | spatie/laravel-permission | `^5.11`/5.11.0 |
| Firebase server SDK | kreait/laravel-firebase | `^5.2`/5.2.0 |
| Payment SDKs — declared, checkout disabled | razorpay `2.*`/2.8.5, stripe-php `^15.0`, braintree_php, paypal-payouts-sdk, paytmchecksum | — |
| RDBMS (config/database.php) | MySQL, `DB_CONNECTION` default `mysql` | dump made on MySQL 5.7.36, Aug 22 2022 |
| Frontend build / UI | laravel-mix, bootstrap, sass | `^6.0.6`, `^5.1.3`, `^1.32.11` |

Also laravel/ui `^4.0`, google/apiclient `^2.15`, guzzle `^7.0.1`, unpinned firebase JS compat SDK (`views/partials/firebase_client.blade.php`). No React/Vite; `tests/` is the stock phpunit `^10.0` skeleton.

## Build, run, deploy
- Prereqs: PHP 8.1+, Composer 2, Node/npm, MySQL 5.7+, a service-account JSON at `FIREBASE_CREDENTIALS`, and the sibling `../../Firebase Import Export Collections/payment_boundary.json`.
- Setup: `composer install`; `npm install`; `cp .env.example .env`; `php artisan key:generate`.
- **Do not bootstrap the DB with `php artisan migrate`.** `database/migrations/` is stock Laravel plus spatie's `create_permission_tables`; nothing there creates `users.role_id` or the custom `roles`/`permissions` columns the app queries. Import `../vibemycar_admin_database.sql`, or every module route 403s.
- Assets `npm run production`; `npm run watch` while iterating. Local serve `php artisan serve`.
- Deploy: no Dockerfile/CI. Root `.htaccess` rewrites into `public/` then `server.php` — Apache + mod_rewrite with the **project root** as document root, not `public/`; hosts pointed at `public/` bypass it. Live target **unverified**.

## Architecture
Laravel 10 monolith, pure admin control plane. MySQL holds only `users`, `roles`, `permissions` plus stock Laravel tables. The 31 controllers mostly run `auth`/`permission` middleware and `return view(...)`; each Blade view (28 folders) embeds the Firebase Web SDK and reads/writes Firestore **directly from the browser** — Laravel never proxies app data. Bridge: `GET /firebase/token` → `FirebaseTokenController::issue` mints a custom token for uid `admin_<user_id>` with claims `{admin:true, role:'admin'}`, then the layout calls `signInWithCustomToken()`. Field-level authorization is Firestore Rules, outside this repo. Payment secrets are the one server-kept exception: `config/payment_boundary.php` reads the sibling registry JSON (see Build) and maps each provider's `secretFields` to `VMC_PAYMENT_*` env names.

## Naming conventions
- Controllers PascalCase + `Controller`; Blade folders snake_case (`resources/views/vehicle_brand/`).
- Route names dot.kebab (`settings.payments.razorpay`); permission pair `<module>,<route-key>` (`permission:sos,sos.edit`).
- Wire-contract paths dot.path lowerCamel.

## Data types & models
MySQL, defined only in `../vibemycar_admin_database.sql` (see Build):
- `users`: id, name, email, email_verified_at, password, **role_id:int(15)**, remember_token, timestamps — `role_id` is absent from `2014_10_12_000000_create_users_table.php`.
- `roles`: id, role_name:varchar, timestamps. `permissions`: id, role_id:int, permission:varchar, routes:varchar, timestamps. Models app/Models/{User,Role,Permission}.php.

Payment boundary (config/payment_boundary.php, PaymentCredentialService), schemaVersion 2: `checkoutAdapters{provider → {adapter, disabledCode | requiredPublicFields}}`, `documents{payment: 19 providers, applePay: 1}`, `credentials{doc → provider → field → env}`, `privateCollection: server_payment_credentials`.

Wire contracts (`resources/contracts/vibemycar_wire_contracts.json`, schemaVersion 1):
- 6 `settings/<docId>` docs: experienceConfig, globalKey, providerConfig, notificationCommandConfig, releaseFeatures, vehicleExperience. Fields e.g. `releaseFeatures.features.{booking,wallet,sos,evCharging}`, `seatLock.{ttlSeconds,maxSeatsPerBooking}`.
- 3 catalogs — `vehicle_catalog`{vehicleCatalogId,make,model,modelYear:int,seatLayoutId,asset3DId}, `vehicle_seat_layout`{seats[]:{seatId,row,column,isDriver}}, `vehicle_asset_3d`{glbUrl,sha256,byteLength:int}.

## API surface
| Operation | Path | Response | Auth | Defined in |
|---|---|---|---|---|
| Mint Firebase custom token | `GET /firebase/token` | `{token, uid}` | `auth` | FirebaseTokenController.php |
| Sanctum user | `GET /api/user` | user JSON | `auth:sanctum` | routes/api.php |
| Legacy checkout, fail-closed (10 routes) | `payments/*`, `payment/*` | 410 `LEGACY_ADMIN_CHECKOUT_REMOVED` if the provider has a central adapter (`razorpay`, `strip`), else 503 `SERVER_CONFIRMATION_UNAVAILABLE` | `auth`+`throttle:10,1` | PaymentController.php |
| Payout, fail-closed | `POST pay-to-user`, `check-payout-status` | 503 | same | UserController.php |
| Notify / email / delete Firebase user | `POST send-notification`, `send-email`, `delete-user/{id}` | redirect/JSON | **no `permission:` wrapper** | Notification/SendEmail/DeleteUserAuthentication controllers |
| Gateway config screens (×~20) + CRUD modules: users, roles, vehicle brand/model/type, tax, currency, cms, faq, on-board, complaints, sos, support, reports | `GET settings/payments/<provider>`, module routes | Blade view | `permission:<module>,<route-key>` | SettingsController.php, routes/web.php |

Firestore names are mixed snake_case/lowerCamel from the source template. 28 `.collection('…')` literals in views = 27 real collections (`users`, `driver_users`, `booking`, `bookedUser`, `documents`, `settings`, `sos`, `thread`, `wallet_transaction`, `vehicle_brand`, …) plus `tmp`, a client-side doc-ID idiom; enumerate by grepping `.collection(` in `resources/views`.

## Security boundary
- CORS (`config/cors.php`): `paths=['api/*','sanctum/csrf-cookie']`, `allowed_methods/_origins/_headers=['*']`, `supports_credentials=false`, `max_age=0`.
- Admin identity: Laravel session guard over MySQL `users`. `PermissionMiddleware` (alias `permission` in `app/Http/Kernel.php`) queries `Permission::where('role_id', $user->role_id)`; unauthenticated → redirect `login`, unauthorized → `abort(403)`.
- Secret **names** only, from `.env.example`: 23 `VMC_PAYMENT_<PROVIDER>_*`, 10 `FIREBASE_*`, `GOOGLE_MAP_KEY`, `APP_KEY`, plus stock `DB_*`/`MAIL_*`/`PUSHER_*` and 5 `AWS_*` (no aws-sdk installed). `server_payment_credentials` is the private Firestore collection; the browser gets only `configured` booleans. `.env` is git-ignored, untracked.

## Known gaps & risks
- **Real FCM legacy server-key literal hardcoded in `config/constant.php`** (`'server_key' => '<literal>'`, not `env()`), committed to the product repo — rotate and move to env.
- Wildcard CORS origins/headers on `api/*` and `sanctum/csrf-cookie`.
- **spatie/laravel-permission is declared and its migration ships, but nothing uses it at runtime.** `PermissionMiddleware` imports `Spatie\Permission\Exceptions\UnauthorizedException`, never throws it, and queries `App\Models\Permission`. The production dump has no `model_has_roles`/`role_has_permissions` tables, and the custom schema (`role_id`,`permission`,`routes`) is incompatible with spatie's (`name`,`guard_name`).
- `ApiKeyAuth.php`/`ApiSecureKeyAuth.php` reference undefined classes (`BaseApiController`, `UserAccessToken`), are unregistered in `Kernel.php`, and compare an inbound `apikey` header to `env('APP_KEY')` — the encryption key reused as an API credential. Dead; delete.
- `send-notification`, `send-email`, `delete-user/{id}` have no `permission:` middleware; `delete-user` reaches `FirebaseService::deleteUser`.
- `config/payment_boundary.php` runs at config-load with `file_get_contents` + `JSON_THROW_ON_ERROR` on a path **outside** the repo. If that sibling directory is missing or its JSON malformed, every request — including the login page — dies during bootstrap. Likeliest, least obvious fresh-clone failure.
- All app data flows browser→Firestore: no server-side validation, audit trail, or rate limiting.
