# Rive official live sources

Read the exact runtime page for the target. Do not preload the index or copy these
pages into the skill.

| Need | Official URL | Crawl context |
|---|---|---|
| Discover an unlisted Rive page | https://rive.app/docs/llms.txt | Search the index for the exact feature and runtime, then open that `.md` page only. |
| Product and interaction model | https://rive.app/docs/getting-started/introduction.md | Confirm when Rive design, animation, state machines, data binding, scripting, and runtime publishing fit the requirement. |
| Flutter runtime | https://rive.app/docs/runtimes/flutter/flutter.md | Verify the current package, initialization, widgets/controllers, loading, errors, disposal, renderer, and troubleshooting. |
| Flutter native renderer bridge | https://rive.app/docs/runtimes/flutter/rive-native.md | Verify platform support, native library behavior, feature support, troubleshooting, build, and test requirements. |
| Apple runtime | https://rive.app/docs/runtimes/apple/apple.md | Verify SwiftUI/UIKit setup, playback, frame rate, semantics, threading, logging, and resource behavior. |
| Android runtime | https://rive.app/docs/runtimes/android/android.md | Verify the current Compose and legacy API boundary, dependency setup, loading states, logging, and examples. |
| Web JavaScript/WASM runtime | https://rive.app/docs/runtimes/web/web-js.md | Verify package installation, canvas setup, instance parameters, loading/caching, resize, cleanup, and advanced resources. |
| Web renderer/package choice | https://rive.app/docs/runtimes/web/canvas-vs-webgl.md | Compare WebGL2, Canvas, lite/single variants, shared/offscreen contexts, bundle size, feature support, and deprecation status. |
| Flutter runtime source/examples | https://github.com/rive-app/rive-flutter | Inspect the current README, example app, issues, and tagged source only when implementation detail is required. |
| Apple runtime source/examples | https://github.com/rive-app/rive-ios | Inspect supported platforms, examples, migrations, issues, and tagged source. |
| Android runtime source/examples | https://github.com/rive-app/rive-android | Inspect supported versions, Compose/legacy examples, build options, issues, and tagged source. |
| Web runtime source | https://github.com/rive-app/rive-wasm | Inspect JavaScript/TypeScript/WASM source and examples only when the runtime docs are insufficient. |
| Current Flutter package | https://pub.dev/packages/rive | Resolve the project-compatible current version and platform metadata; confirm against `pubspec.lock`. |
| Current WebGL2 package | https://www.npmjs.com/package/@rive-app/webgl2 | Resolve the project-compatible current version; confirm against the lockfile. |

Official authority is `rive.app`, `pub.dev`, `npmjs.com`, and official
`github.com/rive-app/*` repositories. Migration pages are compatibility evidence,
not the primary API source.
