# SKILL_ROUTES — vibemycar-web-panel

| Condition | Skill URL |
|---|---|
| Touching PHP, Blade, routing, middleware, artisan, config caching, Composer or anything else framework-level | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/backend-php-laravel.md |
| Touching Firestore, Firebase Auth, App Check, Messaging (FCM), or the client Firebase JS SDK | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/firebase-platform.md |
| Touching Razorpay order creation, signature verification, or checkout | https://raw.githubusercontent.com/ajaitech/model-skills/main/tech/payments-india-razorpay.md |
| Ad-hoc machine or system task, not product code | https://raw.githubusercontent.com/ajaitech/model-skills/main/machine/MACHINE_INDEX.md |

Fetch only rows whose condition is true right now.

Deliberately absent: no AWS row — the `s3` disk and SES entries are stock
Laravel scaffolding, never referenced in `app/`, credentials blank in `.env`. No
React/Vite row — Vite, Tailwind and Alpine are declared in `package.json` but
no Blade file loads them (PROJECT_CONTEXT gap 4). Stripe is live here but has
no shared guide; work from `stripe/stripe-php` and the vendor docs.
