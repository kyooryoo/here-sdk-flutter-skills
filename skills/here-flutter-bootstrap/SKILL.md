---
name: here-flutter-bootstrap
description: >
  Create, set up, or configure a HERE SDK for Flutter (4.x) project: local
  plugin (TAR) integration instead of pub.dev, credentials and
  SdkContext/SDKNativeEngine initialization, adding the HereMap widget,
  loading the first map scene, platform config (iOS/Android), Impeller
  workaround, scope, and fixing a blank map. Always load first — every other
  here-flutter-* skill builds on this initialization.
---

# HERE SDK Flutter — Bootstrap

Emit this initialization **exactly once** per app (unless the app intentionally
inits/disposes per widget — see `shared/threading-and-lifecycle.md` §3).
Shared rules: `shared/credentials-and-licensing.md` (keys, Explore vs Navigate,
legal), `shared/threading-and-lifecycle.md` (main-thread + lifecycle law).

## 1. Project prerequisites

- **Android:** minSdk **24**, compileSdk **36** (must match), targetSdk 36,
  OpenGL ES 3.0+ (also on emulators), ABIs armeabi-v7a/arm64-v8a/x86/x86_64,
  NDK **not needed** — **remove `ndkVersion = flutter.ndkVersion`** from
  `android/app/build.gradle`. Kotlin 2.2.20 required (since 4.28.0.0).
- **iOS:** **15.2+**, Xcode ≥ 16.2. After `flutter pub get`, set in Podfile:
  `platform :ios, '15.2'` and in Runner → General → Minimum Deployments → 15.2.
- **Flutter ≥ 3.44.8 / Dart ≥ 3.12.2** (tested: Flutter 3.47.2 / Dart 3.13.2),
  stable channel only.
- **No web, no desktop** — Android + iOS only. React Native/Xamarin wrappers
  are unsupported.
- Dependency constraint `ffi: >=2.0.1` may conflict with other plugins.
- Real device strongly recommended over emulator/simulator for rendering.

Source: `flutter-introduction-requirements`, `flutter-integrate-here-sdk`

## 2. Dependency — local plugin (only supported way)

```text
# NOT on pub.dev. Download SDK package from platform.here.com/portal/sdk,
# unzip the plugin TAR, RENAME the folder to here_sdk, then:
<app>/plugins/here_sdk/          ← plugin root
```

```yaml
# pubspec.yaml
dependencies:
  flutter:
    sdk: flutter
  here_sdk:
    path: plugins/here_sdk
```

Then `flutter pub get`. The SDK auto-merges `INTERNET` +
`ACCESS_NETWORK_STATE` into the Android manifest.

## 3. Initialize the SDK (main isolate, before runApp)

```dart
import 'package:here_sdk/core.dart';
import 'package:here_sdk/core.engine.dart';
import 'package:here_sdk/core.errors.dart';
import 'package:here_sdk/mapview.dart';

void main() async {
  _initializeHERESDK();
  runApp(MyApp());
}

void _initializeHERESDK() async {
  // Needs to be called before accessing SDKOptions to load necessary libraries.
  SdkContext.init(IsolateOrigin.main);

  AuthenticationMode authenticationMode =
      AuthenticationMode.withKeySecret("YOUR_ACCESS_KEY_ID", "YOUR_ACCESS_KEY_SECRET");
  SDKOptions sdkOptions = SDKOptions.withAuthenticationMode(authenticationMode);

  try {
    await SDKNativeEngine.makeSharedInstance(sdkOptions);
  } on InstantiationException {
    throw Exception("Failed to initialize the HERE SDK.");
  }
}
```

- ⚠ There is **no `HereSdk.init()`** — the sequence above is canonical.
- Init **succeeds even with invalid credentials** — verify by calling
  `SearchEngine()` + a search and checking `SearchError.authenticationFailed`.
- Dispose at app exit: `await SDKNativeEngine.sharedInstance?.dispose();`
  then `SdkContext.release();` — engines/`HereMap` die with the shared
  instance; recreate after any re-init.

## 4. Add the map view

```dart
class _MyAppState extends State<MyApp> {
  late final AppLifecycleListener _appLifecycleListener;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(home: HereMap(onMapCreated: _onMapCreated));
  }

  void _onMapCreated(HereMapController hereMapController) {
    const double distanceToEarthInMeters = 8000;
    MapMeasure mapMeasureZoom =
        MapMeasure(MapMeasureKind.distanceInMeters, distanceToEarthInMeters);
    hereMapController.camera
        .lookAtPointWithMeasure(GeoCoordinates(52.530932, 13.384915), mapMeasureZoom);

    hereMapController.mapScene.loadSceneForMapScheme(MapScheme.normalDay, (MapError? error) {
      if (error != null) {
        print('Map scene not loaded. MapError: ${error.toString()}');
        return;
      }
      // features are RESET on every scene load — re-enable them here
    });
  }
}
```

- `HereMap` is a `StatefulWidget`; the callback delivers a
  `HereMapController` (camera, mapScene, gestures, pick, …).
- Camera can be set **before or after** scene load; until a scene loads the
  map is blank.
- Multiple `HereMap` instances on one screen are supported.

## 5. Impeller workaround (recommended)

HERE SDK "may encounter issues on some devices" with Flutter's default
Impeller (flutter/flutter#152653) — switch to Skia:

```xml
<!-- AndroidManifest.xml, inside <application> -->
<meta-data android:name="io.flutter.embedding.android.EnableImpeller" android:value="false" />
```

```
<!-- ios/Runner/Info.plist -->
<key>FLTEnableImpeller</key>
<false/>
```

⚠ Only the `hello_map_app` example uses this — do not blindly copy manifests
from other examples.

## 6. Render mode (Android glitches)

`HereMapOptions.renderMode` / `MapRenderMode`:
- `surface` — best performance, default; may glitch on Android 12/13 and
  freezes during navigation on some Google phones.
- `texture` — smoother, more overhead; use as the glitch/freezing fix.
- Hybrid composition (`HereMap(NativeViewMode.hybridComposition)`) — fixes
  orientation-change glitches and Android 13–15 lock/unlock black views.
- iOS: modes have no effect.

Details: `here-flutter-troubleshooting`.

## 7. Multiple apps using one credential set

```dart
sdkOptions.scope = "hrn:here:authorization::olp-here-sdk-navigate:project/xxxx";
```

⚠ Set **before** init; wrong/unknown scope ⇒ all auth fails; a scope requires
manually linking services (+ OCM catalogs for Navigate) or requests 403.
See `shared/credentials-and-licensing.md` §5.

## 8. CarPlay / Android Auto (quick facts)

- "Currently, the Flutter platform provides no direct support for Apple's
  CarPlay, Android Auto and Android Automotive" — integration requires
  platform code + method channels; compatibility **not guaranteed** by HERE.
- Android: AndroidX Car App Library, `MapSurface` registered under shared map
  ID ↔ Dart `HereMapController(1)`; iOS: `com.apple.developer.carplay-maps`
  entitlement. SDK initialized once in Dart; native side never re-inits.
- Example: `hello_map_auto_carplay_app`. Full details:
  <https://docs.here.com/here-sdk/docs/flutter-android-auto.md>

## Verify before delivering

- [ ] Plugin folder renamed to `here_sdk`, path dependency in `pubspec.yaml`,
      `flutter pub get` succeeds (never `pub add here_sdk`)
- [ ] `SdkContext.init(IsolateOrigin.main)` → `AuthenticationMode` →
      `SDKOptions` → `await SDKNativeEngine.makeSharedInstance(...)` before `runApp`
- [ ] `loadSceneForMapScheme` called with an error branch; `MapFeatures`
      re-enabled after every scene load
- [ ] Android: compileSdk 36, minSdk 24, `ndkVersion` line removed;
      iOS: Podfile + target at 15.2
- [ ] Impeller disabled (or a documented reason not to)
- [ ] Dispose strategy consistent (`AppLifecycleListener` + `dispose()`);
      engines recreated after any re-init
- [ ] No real credentials committed; Explore/Navigate license checked for the
      features used (`shared/credentials-and-licensing.md` §4)
- [ ] Required privacy text + `HERE_NOTICE` + visible watermark handled
- [ ] Blank-map triage done: credentials/internet/**device time** →
      `loadScene` called? → OpenGL ES 3.0 (simulator: Renderer Maximum) →
      Impeller off → see `here-flutter-troubleshooting`
