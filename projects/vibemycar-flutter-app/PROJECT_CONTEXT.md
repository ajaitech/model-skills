# VibeMyCar Flutter App

## Goal
VibeMyCar is a car-pooling app ("share the drive, split the cost") for riders
and drivers to publish and book shared rides. A driver plans a route in a
multi-step wizard, sets a price per seat, and publishes it; riders search,
book a seat, and pay in-app. The platform takes a 10% service fee on wallet
top-ups, settled via Razorpay. Secondary features: in-ride group chat, a
collaborative ride playlist (Spotify/Amazon Music), a social feed, safety/SOS
tooling, EV-charging lookup. Users are riders/drivers in India (INR default,
Razorpay-only checkout).

## Core requirements
- Publish wizard must never dead-end — always retryable (`add_your_ride_controller.dart`).
- Checkouts run through Razorpay only; 10% fee is additive, integer-minor-unit
  math (`lib/domain/payment/service_fee.dart`).
- Firestore `booking/{id}` is the source of truth for ride lifecycle (`placed`
  -> `onGoing` -> `completed`/`canceled`); group chat mirrors it.
- Every network/Firestore failure shows a retryable banner or named empty
  state, never a blank screen (`RetryBanner`, `InlineStatusBanner`).
- App Check mandatory on non-web platforms; startup never hard-fails on
  optional/decorative remote config (`vibemycar_startup_bootstrap.dart`).
- Interactive controls standardize on the shared `Pressable` widget.

## Tech stack
| Layer | Technology | Version (exact) | Source of truth |
|---|---|---|---|
| Language/SDK | Dart | `>=3.12.2 <4.0.0` | `pubspec.yaml:22` |
| Framework | Flutter | `>=3.44.8` | `pubspec.yaml:23` |
| State mgmt | GetX (`get`) / theme via `provider` | `^4.7.3` / `^6.1.5+1` | `pubspec.yaml:60,73` |
| Backend | Firestore/Auth/Storage/Messaging/App Check | `^6.7.1`/`^6.3.0`/`^4.6.0`/`^13.2.0`/`^16.1.3`/`^0.4.2` | `pubspec.yaml:40-48` |
| Payments | Razorpay | `razorpay_flutter ^1.4.1` | `pubspec.yaml:74` |
| Music | Spotify PKCE, Amazon Music | `spotify_sdk ^4.0.0-dev.2` | `pubspec.yaml:94` |
| Maps | Google Maps/Places, Geolocator | `^2.16.0`/`^0.6.0`/`^14.0.3` | `pubspec.yaml:59,61,63` |
| Auth providers | Google Sign-In / Sign in with Apple | `^7.2.0`/`^8.1.0` | `pubspec.yaml:64,76` |
| Android | AGP/Kotlin/Gradle wrapper/google-services | `9.1.0`/`2.4.10`/`9.6.1`/`4.5.0` | `settings.gradle.kts:22-24` |
| Android target | `minSdk` 24, Java/Kotlin JVM 17, appId `com.vibemycar.aivibe` | — | `app/build.gradle.kts:16,21-23,27,28,58` |
| iOS | Deployment target | `15.5` | `ios/Podfile:2` |

## Architecture
`lib/main.dart` boots via `VibeMyCarStartupGateway` (Firebase + App Check +
language, then required config) before `runApp`. Root is `GetMaterialApp`
wrapped in `ChangeNotifierProvider<DarkThemeProvider>` for theme. No named
routes: navigation is imperative `Get.to(Widget, arguments:{...})`; each
screen's GetX controller reads `Get.arguments` in `onInit`. Screens live in
`lib/app/<feature>/` paired with a controller in `lib/controller/`; newer
features add a pure-logic `lib/domain/<area>/` layer plus an I/O
`lib/services/<area>/` layer (e.g. `domain/payment/service_fee.dart` +
`services/payment/payment_reconciliation_service.dart`). Firestore access is
centralized in `lib/utils/fire_store_utils.dart`. Deploy: Android (Play Store), iOS (App Store), Flutter web (`web/`) — one
Firebase project, `vibemycar` (`416261669348`).

## Naming conventions
- Files `snake_case.dart`; widgets end `Screen`, controllers end `Controller`
  (`AddYourRideController`, `EVChargingController`).
- Firestore collections: `snake_case` constants in `lib/constant/collection_name.dart`
  (`"user_search_history"`, `"ride_group"`, `"vibe_id_claims"`).
- Firestore/JSON fields: `camelCase` (`pricePerSeat`, `bookedSeat`,
  `adminCommission` — `lib/model/booking_model.dart`).
- Env vars (`--dart-define`): mostly `VMC_`-prefixed SCREAMING_SNAKE_CASE;
  exception `SPOTIFY_CLIENT_ID`.
- Git branches: `feat/<slug>` (e.g. `feat/flutter-elite-modernization`).
- Design tokens: 5-token Liquid Glass palette (`app_them_data.dart`) —
  `glassBaseLight/Dark`, `inkLight/Dark`, `actionPrimary`, `actionInvert`,
  `specular`; legacy names re-point onto these five.

## Data types & models
| Entity | Fields (name : type) | Store | Defined in |
|---|---|---|---|
| UserModel | id/firstName/lastName/email/phoneNumber/walletAmount:String; isVerify:bool; vibeId,tenantId/organizationId/appId:String | `users` | `model/user_model.dart` |
| BookingModel | id/createdBy/status:String; publish:bool; pricePerSeat/totalSeat/bookedSeat:String; pickupLocation/dropLocation:Location; stopOver:List\<CityModel\>; vehicleInformation:ref | `booking` | `booking_model.dart` |
| BookedUserModel | id/bookedSeat/subTotal:String; paymentStatus:bool; stopOver:StopOverModel; adminCommission:AdminCommission | `booking/{id}/bookedUser` | `booking_model.dart:175` |
| VehicleInformationModel | vehicleBrand/vehicleModel:ref; licensePlatNumber:String | `user_vehicle_information` | `vehicle_information_model.dart` |
| ReviewModel | bookingId/senderId/receiverId/rating | `review` | `review_model.dart` |
| RazorpayModel | accountId:String; isWithdrawEnabled:bool | `settings/payment` | `payment_method_model.dart` |
| AdminCommission | enable:bool; type/amount:String | `settings/adminCommission` | `booking_model.dart` |

## API surface
| Operation | Trigger / Path | Request/Response | Auth | Defined in |
|---|---|---|---|---|
| Publish ride | Firestore set `booking/{id}` | `toJson()` -> bool | uid==`createdBy` | `add_your_ride_controller.dart:1133` |
| Ride/passenger listeners | `snapshots()` `booking/{id}`, `.../bookedUser` | stream model(s) | rules-gated | `published_details_controller.dart:111,135` |
| Load commission (private) | get `settings/adminCommission` | model or null on failure | authenticated | `fire_store_utils.dart:771` |
| Create payment session | HTTPS payment API | amount/purpose -> `{orderId,keyId}` | bearer, `PaymentApiClient` | `lib/api/api.dart` |
| Razorpay checkout | `Razorpay().open({key,order_id,amount,currency,prefill})` | SDK callback events | key server-issued per session | `select_payment_method_controller.dart:458` |
| Spotify PKCE auth | browser OAuth, redirect `com.vibemycar.aivibe://spotify-callback` | token via exchange client | PKCE, no client secret | `domain/music/spotify_pkce_oauth_flow.dart` |
| EV stations nearby | Google Places API via `http` | lat/lng+radius -> station list | Places API key (config) | `controller/ev_charging_controller.dart` |

## CORS & headers
None configured — Firebase-SDK mobile client, not a web server. Flutter web
(`web/`) relies on Firebase Hosting defaults, absent from this checkout. GAP
if ever served from a non-Firebase origin.

## Security boundary
Auth is Firebase Authentication (phone/Google/Apple). `UserModel` carries
`tenantId`/`organizationId`/`appId` for future cross-app joins, but auth stays
on Firebase under a documented exception (`user_model.dart:31-39`). Secrets
are never hardcoded — resolved via `--dart-define` (names only, see Naming
conventions), with `settings/providerConfig` in Firestore as a remote-config
fallback for the payment API base URL. Android signing reads
`android/key.properties`; `google-services.json`/`GoogleService-Info.plist`
are gitignored (present locally, untracked). Razorpay's checkout key is
server-issued per session, never embedded. `settings/adminCommission`/
`settings/referral` load only for an authenticated uid
(`private_settings_policy.dart`), clearing on failure. No `firestore.rules`
file exists here — rule text unverified; only client-side gating confirmed.
App Check is active on all non-web platforms.

## Known gaps & risks
- No `firestore.rules` in repo — server-enforced rules unverified. GAP.
- Owner defects 1-5 (publish screen empty; payment-method screen empty;
  first-run errors instead of welcome; commission hard-blocking publish; EV
  charging opening unasked/no permission/wrong location) are **not
  reproducible in current source**; each site carries an anti-regression
  comment naming the historical bug and fix: zero-height collapse fixed via
  `IntrinsicHeight` (`published_details_screen.dart:1148-1152`);
  `PaymentFlowShell` "imported by nothing until now"
  (`select_payment_method_screen.dart:34-36`); missing banner used to `throw`,
  trapping first run (`vibemycar_startup_bootstrap.dart:66-78`);
  `publishRide()` has no `adminCommission` dependency, fee shows 0 when absent
  (`add_your_ride_controller.dart:1133`); `_getPosition()` requests permission,
  `locationDenied` has a dedicated UI state (`ev_charging_controller.dart`).
  Owner's installed build vs. this source: unverified, no on-device check done.
- Owner defect 6 (inconsistent buttons): largely addressed — `Pressable` in
  69 files; 8 remaining raw gesture sites are non-button (keyboard dismiss,
  chat media tap-to-open, dialog backdrop).
- `android/build.gradle.kts:29-35` adds tapandpay to a `stripe_android`
  subproject; no Stripe package exists in `pubspec.yaml` — dead config.
