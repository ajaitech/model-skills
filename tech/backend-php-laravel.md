# Backend — PHP, Laravel

## Applies when
- `composer.json` exists AND requires `laravel/framework`.

## Authoritative sources
| Need | URL |
|---|---|
| Framework docs | https://laravel.com/docs |
| Framework repo | https://github.com/laravel/framework |
| PHP language docs | https://www.php.net/docs.php |
| Firebase bridge (server-side custom tokens) | https://github.com/kreait/laravel-firebase |
| Firestore PHP client | https://github.com/googleapis/google-cloud-php |

## Non-obvious rules
- **Route-middleware aliasing lives in `Kernel.php`** (`$routeMiddleware` /
  `$middlewareAliases`, or `bootstrap/app.php` on newer Laravel). A route
  referencing `auth:sanctum` or a custom alias enforces nothing if the alias
  isn't registered there — grep `Kernel.php`/`bootstrap/app.php` before
  trusting a route's protection claim.
- **A declared composer dependency is not proof it's enforced.** Real trap
  seen in production: `spatie/laravel-permission` sits in `composer.json`
  while authorization actually runs through a custom `PermissionMiddleware`
  against an incompatible custom schema. Trace the actual middleware stack —
  never assume a package in `composer.json` is wired into the request cycle.
- **Blade-embedded client SDKs mean the browser talks to the datastore
  directly.** A Firebase JS SDK initialized in a Blade view bypasses Laravel
  authorization entirely — Firestore/Storage security rules are the only
  enforcement layer for those calls; audit them independently.
- `kreait/laravel-firebase` mints a **server-side custom token** the browser
  exchanges for a Firebase ID token — treat that mint endpoint as an auth
  boundary needing its own rate limiting and audit logging.
- `google/cloud-firestore` (or the `google-cloud-php` Firestore component)
  called from PHP runs with a service account's full privileges — no
  security-rules layer applies server-side; app code is the only guard.
- **Artisan/migration discipline**: `artisan migrate` on production is a
  deliberate deploy step (`--force`), never auto-run on request/boot.
- **Config caching pitfall**: `config:cache` freezes `.env` reads at cache
  time — a changed env var post-cache is silently ignored until
  `config:clear` + re-cache.
- `.env` is never the source of truth in production — a committed or
  server-local `.env` should not be trusted; production config comes from the
  platform's secret store.
- Laravel 10 (PHP ^8.1, Mix) and Laravel 12 (PHP ^8.2, Vite) coexist in this
  org — check the `laravel/framework` constraint in `composer.json` before
  assuming Mix vs Vite or `Kernel.php` vs `bootstrap/app.php` middleware.

## Production checklist
- [ ] `composer install --no-dev --optimize-autoloader` in the deploy pipeline
- [ ] `config:cache`, `route:cache`, `view:cache` rebuilt on every deploy and
      every env-var change
- [ ] Migration state verified against production (`artisan migrate:status`)
      before and after deploy
- [ ] CORS explicitly configured with named origins in `config/cors.php` —
      never a wildcard
- [ ] Every gateway/webhook route verifies its provider's signature before
      processing the payload
- [ ] `APP_DEBUG=false`, `APP_ENV=production` confirmed at runtime
- [ ] Firebase/Firestore credentials scoped to a dedicated service account,
      not shared across environments

## Never
- Never trust a client-supplied Firebase ID token (or any user-supplied id)
  as a lookup key without server-side verification via the Firebase
  Admin/`kreait` SDK.
- Never assume a `composer.json` entry is actually wired into the request
  lifecycle — verify the middleware/service-provider registration first.
- Never disable TLS verification (`verify => false`, `CURLOPT_SSL_VERIFYPEER`)
  on outbound calls.
- Never ship a simulated/mock payment success path reachable in production.
- Never treat `.env` as the final word on a production secret — confirm the
  platform secret store overrides it.
