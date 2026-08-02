Project: VibeMyCar Admin Panel (vibemycar-admin-panel), Laravel 10 / PHP 8.1-8.2 / MySQL / Blade + vendored Bootstrap 4 / browser-side Firestore.

| Condition | Skill URL |
|---|---|
| Any PHP/Laravel work — routes, controllers, middleware, Blade, config, migrations, artisan, composer | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/backend-php-laravel.md |
| Firebase Auth/Firestore/Storage — config/firebase.php, FirebaseService, the `GET /firebase/token` custom-token bridge, Firestore JS inside Blade, firestore.rules/storage.rules at the monorepo root | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/firebase-platform.md |
| Razorpay settings screen or the payment-boundary/credential wiring (razorpay/razorpay 2.8.5 is installed; `razorpay→razorpayCheckout` is one of only two non-null checkout adapters) | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-india-razorpay.md |
| Deliberately restyling admin screens to the house design language — the shipped theme is Bootstrap 4.0.0, not Liquid Glass, so treat this as a redesign, never as documentation of what is there | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/design-liquid-glass.md |
| Ad-hoc machine or system task, not product code | https://raw.githubusercontent.com/ajaitech/model-skills/main/machine/MACHINE_INDEX.md |

Fetch only rows whose condition is true right now.

Deliberately absent, with the reason:
- **web-typescript-react-vite / web-nextjs-astro** — no React, Next, Astro or Vite. The only Node tooling is a laravel-mix bundle that no Blade view references and that was never built.
- **backend-node-serverless / go-backend-services / backend-python-aws-cdk** — single PHP monolith, no other backend runtime.
- **data-postgres-dynamo** — MySQL only (`DB_CONNECTION` default `mysql`); app data lives in Firestore.
- **aws-platform** — `.env.example` carries 5 `AWS_*` names and `config/filesystems.php` wires the stock `s3` disk, but neither `aws/aws-sdk-php` nor `league/flysystem-aws-s3-v3` is installed, `FILESYSTEM_DRIVER` defaults to `local`, and nothing calls `disk('s3')`. The disk cannot work.
- **payments-paypal** — paypal-payouts-sdk 1.0.1 and braintree_php 6.11.2 are declared and a PayPal settings screen exists, but every PayPal/Braintree endpoint here is fail-closed by design (503 `SERVER_CONFIRMATION_UNAVAILABLE`) and checkout must never be re-enabled in this app. Fetch it only when working in the sibling Web Panel.
- **flutter-dart-mobile / iot-protocol-translation** — not this codebase.
