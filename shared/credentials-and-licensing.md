# HERE SDK for Flutter — Credentials, licensing, and legal (shared core)

Referenced by every `here-flutter-*` skill. Facts sourced from the official
guide pages cited inline; verify against live docs when licensing changes
(`here-flutter-docs-navigation`). Documented SDK version: **4.28.0.0**.

## 1. SDK delivery — NOT on pub.dev (read first)

> "the HERE SDK is only available as a separate download without SDK
> Dependency Management. Artifactory support via Maven / Gradle, CocoaPods or
> pub.dev is not yet available." — `flutter-integrate-here-sdk`

- Download the **HERE SDK (Explore)** or **HERE SDK (Navigate)** package from
  <https://platform.here.com/portal/sdk>. It contains the license file, docs,
  changelog, examples zip, `HERE_NOTICE`, and a `*.tar.gz` **Flutter plugin**.
- Integration: unzip the TAR → **rename the folder to `here_sdk`** → copy to
  `<app>/plugins/here_sdk` → path dependency:

  ```yaml
  dependencies:
    flutter:
      sdk: flutter
    here_sdk:
      path: plugins/here_sdk
  ```

- Dart imports: `package:here_sdk/core.dart`, `core.engine.dart`,
  `core.errors.dart`, `mapview.dart`, `routing.dart`, `search.dart`,
  `navigation.dart`, `traffic.dart`.
- Upgrades = replace the local `plugins/here_sdk` folder + `flutter pub get`
  (never `flutter pub add here_sdk`).
- Explore binary ⊂ Navigate binary: Navigate packages run Explore example
  apps, not vice versa. API reference/changelog are per-license.

Docs: <https://docs.here.com/here-sdk/docs/flutter-integrate-here-sdk.md>,
<https://docs.here.com/here-sdk/docs/flutter-update-version.md>

## 2. Getting credentials

1. Sign up for the HERE Base Plan → <https://www.here.com/get-started/pricing>.
2. Sign in to the HERE platform portal → <https://platform.here.com/portal>.
3. **Access Manager → Apps → Register new app** → Register (creates app ID).
4. **Credentials tab → OAuth 2.0 → Create credentials → Download** →
   `credentials.properties` (user ID, `here.client.id`, token endpoint,
   `Access key ID`, `Access key Secret`).

⚠ The access key ID/secret are shown **only once** — after Close they cannot
be retrieved. Credentials are tied to the **Application ID**, not the package
name.

Docs: <https://docs.here.com/here-sdk/docs/flutter-get-started.md>

## 3. Using credentials in Dart

```dart
import 'package:here_sdk/core.dart';
import 'package:here_sdk/core.engine.dart';
import 'package:here_sdk/core.errors.dart';

void _initializeHERESDK() async {
  SdkContext.init(IsolateOrigin.main);              // must precede SDKOptions use
  AuthenticationMode authenticationMode =
      AuthenticationMode.withKeySecret(accessKeyId, accessKeySecret);
  SDKOptions sdkOptions = SDKOptions.withAuthenticationMode(authenticationMode);
  try {
    await SDKNativeEngine.makeSharedInstance(sdkOptions);
  } on InstantiationException {
    throw Exception("Failed to initialize the HERE SDK.");
  }
}

void _disposeHERESDK() async {
  await SDKNativeEngine.sharedInstance?.dispose();
  SdkContext.release();
}
```

- **Initialization succeeds even with invalid credentials.** Verify by running
  a feature engine (e.g. `SearchEngine`) and checking for
  `SearchError.authenticationFailed` (log line:
  `Failed to get the authentication token: Authentication failed`).
- Do not hardcode real secrets in committed code; keep them in local config /
  CI secrets and rotate if leaked.

## 4. Explore vs Navigate licenses

| Capability | Explore | Navigate |
|---|---|---|
| Map rendering, schemes, gestures, markers, styles, custom raster layers | ✅ | ✅ |
| Online search/geocoding/suggest/categories, search-along-route | ✅ | ✅ |
| Online routing (car/truck/pedestrian/EV/bus/taxi/scooter, alternatives, tolls) | ✅ | ✅ |
| Traffic flow/incident layers, `TrafficEngine`, `calculateTrafficOnRoute` | ✅ | ✅ |
| Turn-by-turn navigation (`Navigator`/`VisualNavigator`), voice guidance | ❌ | ✅ |
| HERE Positioning (`LocationEngine`), background updates, `ConsentEngine` | ❌ | ✅ |
| Offline maps / `OfflineSearchEngine` / `OfflineRoutingEngine` / prefetch | ❌ | ✅ |
| Warners, lane guidance, Electronic Horizon, truck guidance, `MapMatcher` | ❌ | ✅ |
| Indoor/venue maps, custom catalogs (BYOD), `SegmentDataLoader` | ❌ | ✅ |
| `topoDay`/`topoNight` schemes, `DynamicRoutingEngine`, waypoint optimization | ❌ | ✅ |

- Navigate = signed HERE contract (quote must be accepted with the App ID
  **before** credentials work); Explore = Base Plan self-service, works
  immediately.
- Navigate-only pages carry an explicit banner, e.g. *"Navigation is only
  available with the Navigate license."*, *"Offline maps are only available
  with the Navigate license."*, *"Positioning is only available with the
  Navigate license."*
- Extra entitlements beyond Navigate: EV Charge Points API v3 (for
  `EVSearchEngine`/pool details → `SearchError.forbidden`), VDR license,
  HD GNSS license, TripAdvisor images, truck amenities.
- Sources: <https://docs.here.com/here-sdk/docs/flutter-introduction-editions.md>,
  <https://docs.here.com/here-sdk/docs/flutter-introduction-feature-list.md>

**Rule for agents:** before writing code that uses navigation, positioning,
offline, indoor, or warner APIs, confirm the user has a Navigate license;
otherwise offer the Explore alternative (online engines, app-side handling).

## 5. Multiple apps / scope

One credential set + one app → no scope needed. Multiple apps → create a
project in the HERE platform **Projects Manager**, copy the project HRN, set
it **before init**:

```dart
sdkOptions.scope = "hrn:here:authorization::olp-here-sdk-navigate:project/abcd123";
```

- ⚠ Setting a scope **disables access to all other HERE services** — link the
  needed services in project settings (Routing, Search-*, Vector Tile, …) or
  every request fails with `403: Not authorized to access the resources`.
- ⚠ Navigate + scope: also link the **Optimized Client Map** catalog
  (`hrn:here:data::olp-here:ocm`) plus `ocm-patch` — manual linking only
  needed when a scope is set.
- Unknown scope → all authentication fails (app log only).

Docs: <https://docs.here.com/here-sdk/docs/flutter-multiple-apps.md>

## 6. Legal requirements (must be in every generated app)

- **Privacy policy must include this exact text** (telemetry statement):

  > "Note: This application uses the HERE SDK. Technical, non-personal
  > telemetry data related to map rendering and data usage may be processed by
  > HERE to improve the HERE SDK and map performance."

  Plus disclosure that you license HERE products, that such information is
  collected from your app during end-user use, and that HERE collects and
  processes it.
- **`HERE_NOTICE` file** ships in the SDK package — embed its copyright/OSS
  notices in the app or host and link to them. Required.
- **HERE logo/watermark must stay visible** on the map view. Reposition with
  `setWatermarkLocation()` only when UI requires it; removal requires a special
  agreement with HERE. (Auto-hidden only when both map edges < 250 dp.)
- **Usage stats** (local, session-only):
  `SDKNativeEngine.sharedInstance!.enableUsageStats(true);` — call **before**
  each init (not persisted); read via `.sdkUsageStats`.
- Privacy manifest for iOS: `PrivacyInfo.xcprivacy` ships inside
  `heresdk.xcframework` (required for pre-4.18.2.0: add manually).

Docs: <https://docs.here.com/here-sdk/docs/flutter-about.md>,
<https://docs.here.com/here-sdk/docs/flutter-billing.md>
