# Firebase Platform

## Applies when
- `firebase.json`/`.firebaserc` at repo root.
- `firestore.rules`, `firestore.indexes.json`, or `storage.rules` present.
- `functions/` with a `firebase-functions` dependency.
- `google-services.json` / `GoogleService-Info.plist` present.
- Deps on `firebase`, `firebase-admin`, `firebase_core`, `firebase-tools`.

## Authoritative sources
| Need | URL |
|---|---|
| Product docs | https://firebase.google.com/docs |
| API reference | https://firebase.google.com/docs/reference |
| Firestore | https://firebase.google.com/docs/firestore |
| Security Rules | https://firebase.google.com/docs/rules |
| Auth | https://firebase.google.com/docs/auth |
| Storage | https://firebase.google.com/docs/storage |
| Crashlytics | https://firebase.google.com/docs/crashlytics |
| Remote Config | https://firebase.google.com/docs/remote-config |
| Emulator Suite | https://firebase.google.com/docs/emulator-suite |
| CLI reference | https://firebase.google.com/docs/cli |
| Admin SDK | https://firebase.google.com/docs/admin/setup |
| Release notes | https://firebase.google.com/support/releases |

## Non-obvious rules
- **Create vs update asymmetry.** Rules evaluate `create`/`update` separately; guarding only `update` leaves the first write unprotected.
- **Privilege escalation.** `request.resource.data` on `update` is the proposed post-write doc, not a diff. Gate with `.diff(resource.data).affectedKeys().hasOnly([...])` on privileged fields or a client can flip `role`/`isAdmin`.
- **Rules reject, they don't filter.** A `list` rule evaluates against the query's whole result set — under-constrained leaks the collection, over-constrained 400s the query. Mirror the constraint as an actual query filter.
- **`get()`/`exists()` in rules cost real reads, nest to a hard limit of 10** — a latency/billing trap for per-request role lookups. Put role/tenant in custom auth-token claims instead.
- **Composite indexes are not automatic** — range + extra equality/orderBy needs `firestore.indexes.json`. A missing index throws with a console auto-create link; commit that definition.
- **Storage Rules are a separate file/language**; wildcard segments (`{name}`) scope only that segment.
- **`request.time` is server receipt time** — never trust a client timestamp for TTL logic.
- **Emulators don't enforce App Check or prod quotas** — verify App Check against a real project before launch.
- **Auth account linking is off by default** — same email can create separate accounts per provider unless enabled explicitly.
- **Remote Config must degrade to in-binary defaults** on fetch failure — never block first paint on a config read.
- **Crashlytics needs a forced test crash** to appear within minutes on first install; natural crashes can take hours.

## Production checklist
- [ ] No `allow read, write: if true`; rules tested via `firebase emulators:exec`
- [ ] Composite indexes exported and committed
- [ ] App Check enforced on Firestore, Storage, callable Functions in prod
- [ ] Account-linking policy set explicitly
- [ ] Storage rules scoped by uploader path; size/content-type validated in-rule
- [ ] Every Remote Config flag has a safe in-binary default
- [ ] Crashlytics wired with automated CI symbol upload
- [ ] Billing budget alert configured before public launch

## Never
- Never ship `allow read, write: if request.auth != null` as the final rule — authenticated is not authorized.
- Never allow `update` without `hasOnly()` on privilege fields.
- Never call `get()` per-document in a list rule when a denormalized field avoids it.
- Never treat a client API key as secret; always treat Admin SDK credentials as secret.
- Never deploy rules without an emulator test run first.
