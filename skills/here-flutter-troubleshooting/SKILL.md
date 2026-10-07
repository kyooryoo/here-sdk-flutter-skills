---
name: here-flutter-troubleshooting
description: >
  Debug HERE SDK for Flutter problems: blank/white map, init and credential
  failures, cache-lock crashes, rendering glitches (Impeller/render modes),
  build and Gradle/Xcode errors, navigation freezes, logging with LogControl,
  log capture and symbolication on Android/iOS, and how to file a HERE bug
  report. Use whenever a HERE Flutter app shows an error, crash or blank map.
---

# HERE SDK Flutter — Debugging & troubleshooting

Prereqs: `here-flutter-bootstrap`. Pre-check: minimum requirements
(`flutter-introduction-requirements`), known issues in the changelog
(<https://docs.here.com/here-sdk/changelog>). Sibling skill:
`here-flutter-performance` for cost/memory issues.

## 1. Symptom → cause → fix (official table, `flutter-tips-troubleshooting`)

| Symptom / log | Cause | Fix |
|---|---|---|
| "Unsupported Flutter version" build error | Flutter outside 3.44.8–3.47.x | `git checkout <supported>` in Flutter dir, `flutter doctor`, `flutter clean`, rebuild |
| **Blank white map** | invalid credentials / no internet / slow connection / **wrong device time** (breaks auth) | fix creds per Get started; connectivity; correct clock |
| Blank map + `"No map content will be displayed until valid config is provided."` | `loadSceneForMapScheme` never called | call it; handle `MapError` |
| Blank map + `"Could not create root isolate."` | Flutter downgrade artifact (not SDK) | `rm -rf <flutter>/bin/cache/artifacts/engine/ios*`, `flutter clean`, `flutter run` |
| `"GeneratedPluginRegistrant.m: Module 'here_sdk' not found"` | flutter/flutter#43986 | `flutter clean && rm ios/Podfile && flutter build ios` |
| Crash + `"CL_magma - Couldn't load the default shader."` | no OpenGL ES 3.0 | emulator Settings → Advanced → ES OpenGLAPI level → **Renderer Maximum** |
| Blank map on Android Emulator reopen | emulator bug 34.2.13/34.2.14 (issue 345674553) | use emulator ≤ 34.1.20 (stable); also seen on API 28 images |
| Glitches / map fails on Flutter 3.27.x | **Impeller** | disable Impeller → Skia (bootstrap §5) |
| Android graphical glitches / orientation issues / nav freeze (Google phones) | `MapRenderMode.surface` default | `MapRenderMode.texture`, or hybrid composition (`HereMap(NativeViewMode.hybridComposition)`) |
| Black view after lock/unlock (Android 13–15, Flutter 3.32+) | virtual-display platform view (flutter/flutter#172641) | hybrid composition |
| `HereMap` blank on resume after Android back button | Flutter 3.22.0 bug #148662 | upgrade to Flutter ≥ 3.22.2 |
| `"Modifying properties of a view's layer off the main thread is not allowed"` | Flutter 3.24.x iOS bug | Flutter ≥ 3.27.0 |
| `"These credentials do not authorize access"` | invalid/wrong-generation creds | **3.x creds ≠ 4.x; Explore creds ≠ Navigate APIs** |
| Crash `"Exception was thrown in Java and it was not handled."` | app threw inside an SDK callback | guard callback bodies (`shared/threading-and-lifecycle.md` §5) |
| `FAILED_TO_LOCK_CACHE_FOLDER` / `Storage.LevelDB` / `failedToLockCacheFolder` | 2nd engine with same access key ID → cache lock | see §2 |
| `"No toolchains found in the NDK toolchains folder..."` | NDK ↔ Gradle mismatch | upgrade Gradle classpath (e.g. to AGP 4.1.3+) or fix NDK |
| Gradle "version incompatible with Java" (after AS update) | toolchain drift | align `gradle-wrapper` `distributionUrl` + AGP in `settings.gradle` |
| Release R8 `"Missing classes detected while running R8"` | AGP issue 282544776 | add 4 `dontwarn` rules to `android/app/proguard-rules.pro` (see `hello_map_app`) + reference it in `buildTypes.release` |
| iOS release `-embed-bitcode` fails | bitcode deprecated | `ENABLE_BITCODE = 'NO'` in Podfile `post_install` |
| Gray screen, `"dequeueBuffer failed"` | flutter/flutter#35900 | upgrade Flutter |
| Navigation screen freezes | `VisualNavigator` on selected Google phones | `MapRenderMode.texture` (not default) |
| Map freezes across bg/fg transitions | Flutter version / lifecycle mismatch | align supported Flutter; wire `AppLifecycleListener`/`MapViewLifecycleListener` |
| Rendering oddities on specific GPUs | device restrictions | `ambientOcclusion`/`shadows` prohibited on PowerVR Rogue, Adreno 730–740, Mali 32–38; zone layers darken junctions; transparent polylines lose alpha; `terrain3d` → `terrainHillshade` on PowerVR (changelog 4.27.3) |
| Offline storage spikes / huge updates | unmanaged regions | list `getInstalledRegions()`, show sizes, avoid automatic updates |
| Hot restart freezes after touching the map | flutter/flutter#58987 (known issue 4.28) | restart the app instead |

## 2. Cache-lock crash (two engines, same key)

```dart
SDKOptions sdkOptions = SDKOptions.withAuthenticationMode(authenticationMode);
// Defaults to SDKOptionsActionOnCacheLock.waitLockingAppFinish — can hang forever:
sdkOptions.actionOnCacheLock = SDKOptionsActionOnCacheLock.killLockingApp;
await SDKNativeEngine.makeSharedInstance(sdkOptions);
```

- Values: `killLockingApp` (destroy locking process; safe no-op if unlocked),
  `noAction`, `waitLockingAppFinish` (default, may never end).
- Explicit variant: `LockingProcess.getLockingProcessId(sdkOptions)` →
  `LockingProcess.destroyLockingProcess(sdkOptions, maxTimeoutMs)` (⚠ doc
  snippet is Java-flavored — verify against API reference before pasting).
- Root cause: duplicate access key ID → cache path collision; use distinct
  keys per instance (`shared/threading-and-lifecycle.md` §4).

## 3. Logging (`LogControl`)

```dart
SdkContext.init(IsolateOrigin.main);
// MUST be after init and BEFORE SDKNativeEngine init — min level cannot change afterwards
LogControl.enableLoggingToConsole(LogLevel.logLevelInfo);
// LogControl.disableLoggingToConsole();   // silence (not for debug builds)
```

- Levels (hierarchy, ascending): `logLevelInfo`, `logLevelWarning`,
  `logLevelError`, `logLevelFatal` (+ `logLevelOff`); works in release builds.
- All SDK messages are prefixed **`hsdk-`**.
- Custom sink: implement `LogAppender` → `LogControl` — full runtime control
  (can change level after init); combine with `disableLoggingToConsole()`
  for appender-only output.

## 4. Capturing logs & symbolication

- **Android:** `adb logcat -c` before repro, then
  `adb logcat -t '06-30 16:30:00.820' > log.txt`. Symbolicate:
  `adb logcat -d > crash.txt` + `.so` symbols from the SDK release's
  `debug-symbols` ZIP → `ndk-stack -sym <arch-dir> -dump crash.txt`
  (**use NDK 24.0.8215888 for this release**). Google Play Console: upload the
  debug-symbols ZIP (App bundle explorer → Assets) for deobfuscated reports.
- **iOS:** Xcode → Window → Devices and Simulators → View Device Logs;
  simulator: `xcrun simctl spawn booted log stream --level debug`. dSYMs ship
  inside `heresdk.xcframework` (since 4.13.3) — App Store crashes symbolicate
  via **Open in Xcode** in Connect (also works with Firebase Crashlytics).
  Use only symbols from the exact integrated release.

## 5. Error enums you'll see

`InstantiationException` (engine ctor), `SearchError`
(`authenticationFailed`, `forbidden`, `requestLimitReached`,
`layersNotDownloaded`, `mapNotReady`), `RoutingError` (`invalidParameter`,
`couldNotMatchOrigin`, `noRouteHandle`, `incompatibleRouteHandle`),
`TrafficQueryError`, `MapLoaderError` (`notReady`, `internalError`),
`LocationEngineStatus` (`MISSING_PERMISSIONS`,
`PRIVACY_NOTICE_UNCONFIRMED`, `notSupported`, `notAllowed`),
`MapError`, `ElectronicHorizonErrorCode`, `IndoorRoutingError` (deprecated
members → now `IndoorRouteNoticeCode` route notices). Full enums: API
reference / `core.errors.dart` via `here-flutter-docs-navigation`.

## 6. Filing a HERE bug report

Via <https://www.here.com/contact> — always include **SDK version, platform,
license, logs**, plus: issue type (crash log vs expected/actual), regression
yes/since when, exact repro steps, reproducible with an
[example app](https://github.com/heremaps/here-sdk-examples)?, APIs +
parameters called, repro rate, OS/IDE/device, map version/catalog, other
plugins+versions, screenshots/video, coordinates or GPX for search/routing
issues. GitHub issues are for **docs/example-app** problems only
(<https://github.com/heremaps/here-sdk-examples/issues/new>).
Support channels: [HERE developer Slack](https://heredev.slack.com),
your HERE representative, SLA report in the portal.

## Verify before delivering (a fix)

- [ ] Symptom matched against §1 before inventing a workaround
- [ ] Version alignment checked (Flutter/Gradle/AGP/Xcode/SDK) — changelog
      known issues reviewed for the integrated version
- [ ] Cache lock vs bad credentials vs missing `loadScene` distinguished
      (each has a different log line)
- [ ] Any pasted snippet with Java-style types verified against the API
      reference (`here-flutter-docs-navigation`)
- [ ] Fix works on a real device/emulator, not only hot-restart state
- [ ] If it's an SDK bug: full report assembled per §6
