# Firebase official sources

Identify the exact Firebase product, client/admin SDK, platform, emulator usage, and installed package version before browsing.

| Need | Official URL | Crawl context |
|---|---|---|
| Product implementation | https://firebase.google.com/docs | Open only the exact product and platform guide; preserve production/emulator separation. |
| API signatures | https://firebase.google.com/docs/reference | Select platform and SDK version; verify types, return values, errors, and availability. |
| Admin/server integration | https://firebase.google.com/docs/admin/setup | Follow the target runtime and credential model; never place service credentials in clients. |
| Cloud Functions | https://firebase.google.com/docs/functions | Verify generation/runtime, triggers, regions, deployment, and emulator differences. |
| Compatibility only: SDK and tooling changes | https://firebase.google.com/support/releases | After the product/API reference, review installed-to-target changes and deprecations for the exact SDK or tool. |

Official domains: `firebase.google.com`, `cloud.google.com`, `developers.google.com`, and official `github.com/firebase/*` repositories linked by Firebase.
