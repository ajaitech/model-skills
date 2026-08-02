# Flutter / Dart Mobile

## Applies when
`pubspec.yaml` exists at repo root.

## Authoritative sources
| Source | URL |
|---|---|
| Flutter docs | https://docs.flutter.dev |
| Dart docs | https://dart.dev |
| Flutter engine/SDK repo | https://github.com/flutter/flutter |
| First-party plugins | https://github.com/flutter/packages |
| Package registry | https://pub.dev |
| Android Gradle Plugin | https://developer.android.com/build/releases/gradle-plugin |
| Kotlin releases | https://kotlinlang.org/docs/releases.html |
| iOS signing / Xcode | https://developer.apple.com/documentation/xcode |

## Non-obvious rules
- Gradle, AGP, and Kotlin versions are a matched triple. Bumping one without the others fails the build with opaque Gradle sync errors, not a Flutter-level message — check `android/settings.gradle` + `android/build.gradle` together, never one file in isolation.
- `flutter analyze` only proves the code compiles and lints clean. It proves nothing about runtime UI correctness — a screen can analyze clean and render an empty/broken layout on device. Treat it as a pre-check, never as done.
- State management has no framework default (Provider, Riverpod, Bloc, GetX all ship as plain packages). Detect the project's existing convention from `pubspec.yaml` and match it — do not introduce a second state pattern into an existing app.
- `MethodChannel`/`EventChannel` calls run on the platform thread; blocking work invoked through a channel handler stalls the UI. Offload heavy native work to a background isolate or a native async API, not the channel call itself.
- Hot reload does not re-run `main()`, does not replay `initState`, and does not pick up changed native/platform code, changed `const` values used in `enum`s, or new asset registrations. When a change "doesn't show up," full restart (`R` vs `r`) before assuming a bug.
- iOS provisioning profile UUIDs drift silently after an Xcode upgrade or profile renewal — a signed build that worked yesterday can fail today with no code change. Verify the profile in Xcode's signing pane, not just `flutter build ipa` output.
- Commit `pubspec.lock` for applications (reproducible builds); packages/plugins intended for reuse should NOT commit it (let consumers resolve).
- Dart is sound-null-safe by default since Dart 2.12 — a legacy (`// @dart <2.12`) file mixed into an otherwise-migrated app silently reintroduces nullability holes at that file's boundary.

## Production checklist
- [ ] `flutter analyze` clean, zero warnings suppressed without justification
- [ ] Real device or emulator run, screen driven manually, screenshot taken as proof — no test-suite substitute
- [ ] `flutter build appbundle`/`apk --release` signed with the release (not debug) keystore
- [ ] `flutter build ipa` with a valid, non-expiring-soon provisioning profile
- [ ] App icons, splash screen, and version/build number bumped for this release
- [ ] Crash reporting (e.g. Crashlytics) wired and verified with a test crash
- [ ] R8/ProGuard rules verified against release build (no stripped classes breaking reflection-based plugins)
- [ ] Deep links / App Links / Universal Links tested on-device, not just declared in manifest

## Never
- Never treat a green `flutter test`/widget-test run as evidence a screen works — only a driven, screenshotted device run counts.
- Never ship a release build signed with the debug keystore.
- Never hardcode API keys or secrets directly in Dart source; use platform-level secret injection.
- Never bump Gradle, AGP, or Kotlin independently without checking Flutter's supported version matrix for the other two.
- Never assume an iOS release build will succeed without first confirming the signing certificate and provisioning profile are both currently valid.
