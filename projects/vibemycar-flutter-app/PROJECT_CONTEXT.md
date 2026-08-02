# VibeMyCar Flutter App

## Goal
Car-pooling: drivers publish a route via a multi-step wizard with per-seat
pricing; riders search, book, pay in-app. 10% additive fee on wallet
transactions via Razorpay (India, INR, Razorpay-only). Secondary: group
chat, collaborative playlist (Spotify/Amazon Music), social feed, safety/SOS,
EV-charging lookup.

## Core requirements
- Publish wizard never dead-ends — retry overwrites the same reserved draft id
  (`add_your_ride_controller.dart:1129-1133`).
- 10% fee: additive, integer minor units, ceiling-rounded, never zero on
  nonzero (`domain/payment/service_fee.dart`).
- `booking/{id}` is source of truth for lifecycle `placed` -> `onGoing` ->
  `completed`/`canceled` (`constant.dart:83-86`); chat mirrors it.
- Every failure shows a retryable banner or named empty state, never a blank
  screen (`widgets/inline_feedback.dart:116,210`).
- Startup never hard-fails on decorative config; App Check mandatory non-web
  (`vibemycar_startup_bootstrap.dart:32-40,66-78`).
- Interactive controls use the shared `Pressable` widget.

## Build & run
- Prereqs: `android/app/google-services.json` + `ios/Runner/GoogleService-Info.plist`
  — gitignored (`.gitignore:60-61`), must exist locally.
- `flutter run` needs zero dart-defines; optional `SPOTIFY_CLIENT_ID`,
  `VMC_PAYMENT_API_BASE_URL` (else `settings/providerConfig` fallback),
  `VMC_AMAZON_MUSIC_CLIENT_ID`, `VMC_AMAZON_MUSIC_API_BASE_URL`.
- Firestore env is a code constant: `currentEnv`
  (`fire_store_utils.dart:50`); `staging` targets named DB `staging`.

## Tech stack
| Layer | Tech (exact version) | Source |
|---|---|---|
| SDK | Dart `>=3.12.2 <4.0.0`; Flutter `>=3.44.8` | `pubspec:22-23` |
| State | GetX `^4.7.3`; theme `provider ^6.1.5+1` | `pubspec:60,73` |
| Firebase | Firestore `^6.7.1`, Auth `^6.3.0`, Core `^4.6.0`, Storage `^13.2.0`, Messaging `^16.1.3`, App Check `^0.4.2` | `pubspec:40-48` |
| Payments | `razorpay_flutter ^1.4.1` | `pubspec:74` |
| Music | `spotify_sdk ^4.0.0-dev.2` (PKCE); Amazon Music via HTTP | `pubspec:94` |
| Maps | google_maps_flutter `^2.16.0`, places `^0.6.0`, geolocator `^14.0.3` | `pubspec:59-63` |
| Sign-in | google_sign_in `^7.2.0`; sign_in_with_apple `^8.1.0` | `pubspec:64,76` |
| Android | AGP 9.1.0, Kotlin 2.4.10, google-services 4.5.0, Gradle 9.6.1; minSdk 24, JVM 17, appId `com.vibemycar.aivibe` | `settings.gradle.kts:22-24`; wrapper; `app/build.gradle.kts:16,27-28,58` |
| iOS | deployment target 15.5 | `ios/Podfile:2` |

## Architecture
`main.dart` boots via `VibeMyCarStartupGateway` (language + Firebase + App
Check, then required config, then maintenance/forced-update gates —
`settings/globalValue`.`release`, `fire_store_utils.dart:355`) before
`runApp`. Root: `GetMaterialApp` in `ChangeNotifierProvider<DarkThemeProvider>`.
No named routes — `Get.to(Widget, arguments:{...})`; controllers read
`Get.arguments` in `onInit`. Screens `lib/app/<feature>/` +
controllers `lib/controller/`; newer features add pure `lib/domain/<area>/` +
I/O `lib/services/<area>/`. Firestore access centralizes in
`utils/fire_store_utils.dart`. Deploy: Play Store, App Store, Flutter web —
one Firebase project, `vibemycar`.

## Naming conventions
- Files `snake_case.dart`; widgets end `Screen`, controllers `Controller`.
- Collections: `snake_case` constants in `constant/collection_name.dart`
  (`"ride_group"`, `"vibe_id_claims"`); doc fields `camelCase` (`pricePerSeat`).
- Env vars: `VMC_`-prefixed; sole exception `SPOTIFY_CLIENT_ID`.
- Branches: `feat/<slug>` (e.g. `feat/flutter-elite-modernization`).
- Design tokens: 5-token Liquid Glass palette (`app_them_data.dart`) —
  `glassBase`/`ink` (Light+Dark), `actionPrimary`, `actionInvert`, `specular`;
  legacy names re-point to these five.

## Data types & models
| Entity | Fields | Store | Source |
|---|---|---|---|
| UserModel | walletAmount:String (money-as-string); isVerify:bool; vibeId + tenantId/organizationId/appId | `users` | `user_model.dart` |
| BookingModel | status/createdBy:String; pricePerSeat/totalSeat/bookedSeat:String (numbers-as-string); publish:bool; pickup/drop:CityModel; stopOver:List\<CityModel\>; vehicleInformation embedded | `booking` | `booking_model.dart` |
| BookedUserModel | bookedSeat/subTotal:String; paymentStatus:bool; pickup/drop:Location (NOT CityModel); stopOver:StopOverModel; adminCommission:AdminCommission | `booking/{id}/bookedUser` | `booking_model.dart:175` |
| VehicleInformationModel | licensePlatNumber (sic); brand/model embedded | `user_vehicle_information` | — |
| RazorpayModel | enable/isSandbox/isWithdrawEnabled:bool; razorpayKey legacy, unused by checkout | `settings/payment` | `payment_method_model.dart:298` |
| AdminCommission | enable:bool; type/amount:String | `settings/adminCommission` | `admin_commission.dart` |

## API surface
| Operation | Path / call | Auth | Source |
|---|---|---|---|
| Publish ride | Firestore set `booking/{id}` via `toJson()` | uid==`createdBy` | `add_your_ride_controller.dart:1133` |
| Ride/passenger streams | `snapshots()` `booking/{id}` + `.../bookedUser` | rules-gated | `published_details_controller.dart:111,135` |
| Private settings | get `settings/adminCommission`,`settings/referral`; cleared on failure | authed uid only | `fire_store_utils.dart:771` |
| Payment session | HTTPS API: purpose/amountMinor -> keyId/orderId | bearer = Firebase ID token | `api/external/l2/payment_api_client.dart:465` |
| Razorpay checkout | `_razorpay.open({key,order_id,amount,currency,prefill})` | key server-issued per session | `select_payment_method_controller.dart:468` |
| Spotify PKCE | browser OAuth -> `com.vibemycar.aivibe://spotify-callback` | PKCE, no secret | `domain/music/spotify_pkce_oauth_flow.dart` |
| EV stations | Google Places via `http`: lat/lng+radius -> stations | Places key | `controller/ev_charging_controller.dart` |

## Security boundary
Firebase Auth (phone/Google/Apple); stays on Firebase under a documented
tenancy exception (`user_model.dart:31-39`). Android signing reads
`android/key.properties` (gitignored). Razorpay key server-issued per session,
never embedded. Private settings load only for an authed uid
(`domain/auth/private_settings_policy.dart`). No
CORS/header config — mobile SDK client; web build relies on Firebase Hosting
defaults (config absent here).

## Known gaps & risks
- No `firestore.rules` here — server-enforced rules unverified; only
  client-side gating confirmed. GAP.
- `android/build.gradle.kts:29-35` injects play-services-tapandpay into a
  `stripe_android` subproject; no Stripe package in `pubspec.yaml` — dead
  config. `app/build.gradle.kts:68` pins play-services-wallet 19.4.0.
- Release build unminified/unshrunk (`app/build.gradle.kts:45-46`).

## Owner defects — source-tree verification (2026-08-02)
CAVEAT — SOURCE TREE ONLY, not the owner's installed build: do NOT read this
as "the app is fixed"; proof is still a driven, screenshotted device run.
Defects 1-5 not reproducible in source; each site has an anti-regression
comment:
1. Publish screen empty — `IntrinsicHeight` fix (`published_details_screen.dart:1148-1152`).
2. Payment screen empty — `PaymentFlowShell` "imported by nothing until now" (`select_payment_method_screen.dart:33-36`).
3. First-run error trap — missing banner used to `throw` (`vibemycar_startup_bootstrap.dart:66-78`).
4. Commission hard-block — `publishRide()` has zero `adminCommission` refs (`add_your_ride_controller.dart:1133`).
5. EV — explicit tap only (`home_screen.dart:331`); permission flow (`ev_charging_controller.dart:349-365`); `locationDenied` UI (`ev_charging_screen.dart:437`).
6. Buttons — `Pressable` in 75 files; 9 raw gesture sites, all non-button:
   rating slider, pinch-zoom, keyboard dismiss, 4 chat-media viewers, dialog
   tap-guard, get-started CTA (`get_started_route_gate.dart:188`).

## Owner-blocked (owner's notes 2026-07-31)
| Item | Action | Impact |
|---|---|---|
| Keystore password leaked | Rotate + purge leak | Release-signing integrity at risk |
| Firebase CLI expired | `firebase login --reauth` | Rules/index deploys blocked |
| Gemini spending cap | Raise cap or switch account | 12 car photos missing |
| Launch seed unapplied | Run with `--apply` | 100 profiles + 280 rides absent |

Verification standard: done = running app driven on device + screenshot;
`flutter analyze` catches compile errors only. Emulator: `VibeMyCar_API_36`.
adb: `/opt/homebrew/share/android-commandlinetools/platform-tools/adb`
