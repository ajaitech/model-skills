# Flutter official sources

Match `pubspec.lock`, the Flutter SDK constraint, and `flutter --version`. Keep Flutter and bundled Dart compatibility aligned.

| Need | Official URL | Crawl context |
|---|---|---|
| API signatures | https://api.flutter.dev/ | Verify the exact library, class, member, nullability, and deprecation status. |
| Framework guides | https://docs.flutter.dev/ | Open the exact platform, widget, architecture, testing, or deployment guide. |
| CLI syntax | https://docs.flutter.dev/reference/flutter-cli | Cross-check with local `flutter help` before execution. |
| Compatibility only: breaking changes | https://docs.flutter.dev/release/breaking-changes | Use after the API/guide page when an installed-to-target migration may affect the implementation. |
| Compatibility only: release notes | https://docs.flutter.dev/release/release-notes | Use after the API/guide page only to confirm a changed stable-channel behavior or upgrade delta. |

Official domains: `docs.flutter.dev`, `api.flutter.dev`, and linked `github.com/flutter/*` source.
