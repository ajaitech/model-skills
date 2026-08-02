# VibeMyCar Admin Panel

Role-gated internal back-office for ops/support/finance staff, not riders or drivers. Control plane over the Firestore project the mobile apps use, plus a small MySQL database for admin identity and RBAC.

## Core requirements
- Gate every module route with `permission:<module>,<route-key>` (75 groups, `routes/web.php`). Authorize browser Firestore/Storage only via `GET /firebase/token`, never a standing credential.
- Never re-enable checkout; the sibling Web Panel (Laravel `^12.0`/PHP `^8.2`) owns `POST /api/payment/checkout-sessions`.
- Keep edited documents shaped to `resources/contracts/vibemycar_wire_contracts.json`; mobile clients read them.

## Tech stack
| Layer | Package | Version (constraint / composer.lock) |
|---|---|---|
| Language | php | `^8.1` — **installs only on 8.1.x / 8.2.x** |
| Framework, API tokens | laravel/framework, laravel/sanctum | `^10.0`/v10.13.0, `^3.2`/v3.2.5 |
| Firebase server SDK | kreait/laravel-firebase → kreait/firebase-php | `^5.2`/5.2.0 → 7.2.1 |
| RBAC pkg — declared, **unused at runtime** | spatie/laravel-permission | `^5.11`/5.11.0 |
| Payment SDKs, checkout disabled | razorpay 2.8.5, stripe-php v15.0.0, braintree_php 6.11.2, paypal-payouts-sdk 1.0.1, paytmchecksum v1.1.0 | — |
| RDBMS | MySQL (`DB_CONNECTION` default) | dump on 5.7.36 |

Shipped UI is **Bootstrap 4.0.0** + jQuery 3.5.1 vendored in `public/`; `package.json`'s laravel-mix `^6.0.6` / bootstrap `^5.1.3` / sass `^1.32.11` bundle is referenced by no view and never built. Browser Firebase: pinned **9.23.0** compat SDK + geofire 5.0.1 (`views/layouts/app.blade.php:386-393`).

## Build, run, deploy
- **PHP 8.2.x only.** composer.lock's three kreait packages require `php ~8.1.0|~8.2.0`, so `composer install` aborts on 8.3+; 8.1 is EOL (php.net/supported-versions.php).
- Beyond Laravel's defaults the lock needs curl, dom, fileinfo, libxml, sodium, xmlwriter, `pdo_mysql`; `ext-sodium` is often absent.
- `composer install`; `cp .env.example .env`; `php artisan key:generate`; `php artisan serve`. `npm run production` builds only the dead bundle.
- **Never `php artisan migrate`.** `config/permission.php:39,47` aims spatie's shipped migration at the *same table names* the custom schema uses, with clashing columns, and nothing creates `users.role_id`. Import `../vibemycar_admin_database.sql`, or every module route 403s.
- kreait auto-discovers credentials via `FIREBASE_CREDENTIALS`; `send-notification` needs its own at `storage/app/firebase/credentials.json`.
- `config/payment_boundary.php:5-6` reads `../../Firebase Import Export Collections/payment_boundary.json` at config-load, **outside this repo** — missing or malformed kills every request, login included, at bootstrap. Likeliest fresh-clone break.
- After editing `settings/payments/*.blade.php` run `php tests/payment_boundary_contract_test.php`; it fails if a view touches a registered secret field.
- No Dockerfile, no CI. Root `.htaccess` rewrites into `public/` then `server.php`, so Apache needs the **project root** as document root; hosts pointed at `public/` bypass it.

## Architecture
Laravel 10 monolith, control plane only: 29 feature controllers mostly gate and `return view(...)`, and the 27 Blade folders / 104 views drive Firestore **directly from the browser** via the layout's Firebase Web SDK; Laravel never proxies app data, so there is no server-side validation, audit trail or rate limits. Bridge: `FirebaseTokenController::issue` mints a custom token for uid `admin_<user_id>`, claims `{admin:true, role:'admin'}`; `app.blade.php:397-410` calls `signInWithCustomToken`. **Firestore and Storage Rules are the real authorization boundary**: `firestore.rules:13` and `storage.rules:14` define `isAdmin()` as `request.auth.token.admin == true`, the claim minted here. Payment secrets stay server-side — `AppServiceProvider::boot` view-composes `$paymentBoundary` into `settings.payments.*` as `configured` booleans. Conventions: snake_case Blade folders (except `on-board`), dot.kebab routes, 39 permission modules.

## Data & models
MySQL schema exists **only** in `../vibemycar_admin_database.sql` — 7 tables, none spatie's: `users`(…, **role_id int(15)**, …), absent from `2014_10_12_000000_create_users_table.php`; `roles`(id, role_name); `permissions`(id, role_id, permission, routes). Seeded: 1 role, 94 permission rows, one Super Administrator with a committed bcrypt hash — rotate.

`config/payment_boundary.php` (schemaVersion 2) reads the sibling registry: in `checkoutAdapters` only `strip→stripePaymentSheet` and `razorpay→razorpayCheckout` are set; `documents{payment: 19 providers, applePay: 1}`; `privateCollection: server_payment_credentials` (`firestore.rules:312` denies all client access).

Firestore: 29 distinct `.collection(` literals in `resources/views` = **26 top-level collections**, `bookedUser` (subcollection of `booking/{id}`, also written as the prefix `booking/`) and `tmp`, a client-side doc-ID generator never persisted.

## API surface
| Operation | Path | Response | Auth |
|---|---|---|---|
| Mint Firebase custom token | `GET /firebase/token` | `{token, uid}` | `auth` |
| Sanctum user (all of `routes/api.php`) | `GET /api/user` | user JSON | `auth:sanctum` |
| Legacy checkout, 7 routes | `payments/*` | stripe → 410 `LEGACY_ADMIN_CHECKOUT_REMOVED`; paytm (4) + paypal (2) → 503 `SERVER_CONFIRMATION_UNAVAILABLE` (no adapter) | `auth`+`throttle:10,1` |
| Legacy status, 3 routes | `GET payment/{success,failed,pending}` | 410 always | same |
| Payout, 2 routes | `POST pay-to-user`, `check-payout-status` | 422 on bad fields, else 503 `PAYOUT_*_UNAVAILABLE` | same |
| Notify, email, delete Firebase user | `POST send-notification`, `send-email`, `delete-user/{id}` | JSON | `send-notification` `auth` only; other two **none — CSRF only** |
| 20 gateway screens + CRUD modules | `GET settings/payments/<provider>`, module routes | Blade view | `permission:<module>,<route-key>` |

## Security boundary
- CORS (`config/cors.php`): `paths=['api/*','sanctum/csrf-cookie']`, methods/origins/headers all `['*']`, `supports_credentials=false`.
- Laravel session guard over MySQL `users`. `PermissionMiddleware` (alias `permission`, `Kernel.php:69`, in the deprecated `$routeMiddleware` array) queries `Permission::where('role_id', $user->role_id)` (line 29); unauthenticated → `login`, unauthorized → `abort(403)`. `Auth::routes()` publishes `/register`, but `Auth/RegisterController.php:39-44` redirects to `/`, closing self-registration.
- Env **names** only: 23 `VMC_PAYMENT_*`, 10 `FIREBASE_*`, `APP_KEY`, `GOOGLE_MAP_KEY`, stock `DB_*`/`MAIL_*`/`PUSHER_*`, 5 `AWS_*` (no aws-sdk). `GOOGLE_MAP_KEY` and `FIREBASE_FIRESTORE_DATABASE` are unread; the Maps key comes from `settings/globalKey.googleMapKey`. `.env` is git-ignored.

## Known gaps & risks
- **Every authenticated page 500s.** `app.blade.php:610` calls `route('store-firebase-service')` — defined nowhere, unguarded by any Blade conditional, in the layout 95 views extend. Login renders (standalone view); all else throws `RouteNotFoundException`. Fix first.
- **Real FCM legacy server key hardcoded** as `apikey.server_key` in `config/constant.php` — a literal, not `env()`, committed. Nothing reads it (notifications use FCM HTTP v1): dead config holding a live credential. Rotate it.
- `NotificationController.php:58-59` disables TLS peer/host verification on the FCM call; `.gitignore` covers `/storage/app/private/firebase/*` but the code reads `storage/app/firebase/credentials.json`, so a key left there is tracked.
- **spatie is declared and its migration ships, yet nothing uses it**: `PermissionMiddleware.php:9` imports an exception it never throws, the custom columns clash with spatie's `name`/`guard_name`, and the dump has no `model_has_roles`/`role_has_permissions`.
- `CheckUserRoleMiddleware` runs on **every** web request (`Kernel.php:42`) and dereferences `$users->roleName` off a `first()`; an admin whose `role_id` matches no `roles` row 500s everywhere.
- `ApiKeyAuth`/`ApiSecureKeyAuth` reference undefined classes (`BaseApiController`, `UserAccessToken`), are unregistered in `Kernel.php`, and compare an `apikey` header to `env('APP_KEY')` — the encryption key as an API credential. Dead, as is `app/Exports/UserExport.php`.
- `send-email` mails arbitrary recipients unauthenticated and unvalidated (`SendEmailController.php:17-29`); `delete-user/{id}` reaches `FirebaseService::deleteUser` unauthenticated. CSRF is the only gate.
