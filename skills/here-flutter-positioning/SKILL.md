---
name: here-flutter-positioning
description: >
  Get GPS locations in a HERE SDK for Flutter app: LocationEngine setup,
  Android/iOS permission manifests, runtime permission handling, privacy-notice
  confirmation, location/status/issue listeners, accuracy modes and
  LocationOptions, showing the blue-dot puck, background location updates,
  last-known location, and location simulation for tests. Use for any
  current-location or location-feed task.
---

# HERE SDK Flutter — Positioning

Prereqs: `here-flutter-bootstrap`. **Navigate license only** — every page
carries *"Positioning is only available with the Navigate license."*
Callbacks arrive on the main thread (`shared/threading-and-lifecycle.md`).

## 1. Permissions (exact — Flutter has no built-in handling; use
`permission_handler` for runtime prompts)

**Android — `android/app/src/main/AndroidManifest.xml`:**

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE"/>
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.CHANGE_WIFI_STATE" />
```

- Target SDK 31+: the user must grant **"precise"** location — coarse-only
  makes `LocationEngine` fail with `MISSING_PERMISSIONS`.
- Background (recommended path): foreground service —
  `FOREGROUND_SERVICE_LOCATION` (API 34+) + service with
  `android:foregroundServiceType="location"` (API 31+).
  `ACCESS_BACKGROUND_LOCATION` is **not** required with a foreground service.
- `WAKE_LOCK` optional — SDK uses wake locks only if granted.

**iOS — `ios/Runner/Info.plist`:**

```xml
<key>UIRequiredDeviceCapabilities</key>
<array><string>location-services</string><string>gps</string></array>
<key>NSLocationWhenInUseUsageDescription</key>
<string>This app needs to access your current location to display it on the map.</string>
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>   <!-- background only -->
<string>This app needs to access your current location to display it on the map.</string>
<key>NSMotionUsageDescription</key>
<string>Motion detection is needed to determine more accurate locations, when no GPS signal is found or used.</string>
```

⚠ Having `gps` in `UIRequiredDeviceCapabilities` prevents install on GPS-less
devices (some iPads).

## 2. Engine + listeners (6-step official flow)

1. Add + request permissions. 2. Create `LocationEngine` + listeners.
3. **Android first launch:** inform the user about Wi-Fi/cellular data
   collection and link the HERE Privacy Notice. 4. **Android:** call the
   privacy confirmation **before `start()`** — otherwise **zero locations**:

```dart
locationEngine.confirmHEREPrivacyNoticeInclusion();
// or confirmHEREPrivacyNoticeException();  → on failure: LocationEngineStatus.PRIVACY_NOTICE_UNCONFIRMED
```

5. Start once + set accuracy. 6. Consume `Location` updates.

```dart
try {
  _locationEngine = LocationEngine();
} on InstantiationException {
  throw Exception("Initialization of LocationEngine failed.");
}

class _PosState extends State<...> with LocationListener, LocationStatusListener {
  void startLocating() {
    _locationEngine.addLocationListener(this);
    _locationEngine.addLocationStatusListener(this);
    _locationEngine.startWithLocationAccuracy(LocationAccuracy.bestAvailable);
  }
  void stopLocating() {
    _locationEngine.removeLocationStatusListener(this);
    _locationEngine.removeLocationListener(this);
    _locationEngine.stop();
  }
  @override
  void onLocationUpdated(Location location) { /* main thread */ }
  @override
  void onStatusChanged(LocationEngineStatus s) { /* engineStarted|engineStopped|alreadyStarted|... */ }
  @override
  void onFeaturesNotAvailable(List<LocationFeature> features) {}
}
```

- ⚠ Only **one** `LocationEngine` may be started at a time; check
  `isStarted()`. Cannot be created in `Application.onCreate()`.
- `LocationIssueListener` (via `addLocationIssueListener`) reports error
  causes (e.g. VDR calibration, HD GNSS extrapolation).
- Last known location without starting:
  `Location? last = _locationEngine.lastKnownLocation;` (null until first fix;
  persists across sessions).
- `Location`: coordinates + `time` guaranteed; `bearingInDegrees`, `speed`,
  `horizontalAccuracyInMeters` nullable. **`time` is required** for map
  matching/navigation — set `location.time = DateTime.now();` when building
  locations yourself.
- Accuracy semantics: `horizontalAccuracyInMeters` = CEP68 (CEP99 ≈ 2.01×);
  ⚠ Android altitude = WGS-84 ellipsoid, iOS = mean sea level.

## 3. Accuracy modes & options

`LocationAccuracy`: `bestAvailable`, `navigation`, `subMeterNavigation`,
`tensOfMeters`, `hundredsOfMeters`, `kilometers`.

- Recommended feed for navigation: `LocationAccuracy.navigation` (≈1 Hz).
- Android-only fine control — `startWithLocationOptions(LocationOptions()..
  wifiPositioningOptions.enabled = true .. sensorOptions.vdrEnabled = false ..
  notificationOptions.desiredIntervalMilliseconds = 60000)` — iOS returns
  `LocationEngineStatus.notSupported`. `smallestInterval` is a hard cap,
  `desiredInterval` is best-effort.
- `subMeterNavigation` = **HD GNSS license** + Android 12+, L1+L5 dual-freq
  hardware; **not supported on iOS**. VDR = separate VDR license,
  Android-only, 10 Hz max, needs minutes of calibration.
- iOS-only: `setPauseLocationUpdatesAutomatically()` (auto-pause when
  stationary).

## 4. Show the puck (`LocationIndicator`)

```dart
LocationIndicator puck = LocationIndicator();
puck.isAccuracyVisualized = true;                                  // accuracy halo
puck.locationIndicatorStyle = LocationIndicatorIndicatorStyle.pedestrian;  // or .navigation
puck.enable(hereMapController);
puck.updateLocation(location);
// puck.setHaloColor(...) is BETA
```

- Never scales with zoom; always drawn on top; **not pickable**.
- During guidance let `VisualNavigator` own its indicator — do not fight it
  with a second one.
- Set an initial location (last-known or default) before the first fix so the
  UI isn't empty.

## 5. Background updates

**Android** — implement a **foreground service** (see §1) and expose
`start/stop` from Dart via platform code; API ≤28 needs no changes. The docs
warn the positioning example "may not fully demonstrate foreground service
implementation for all Android versions."

**iOS** — `Info.plist`:

```xml
<key>UIBackgroundModes</key>
<array><string>location</string><string>processing</string></array>
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array><string>$(PRODUCT_BUNDLE_IDENTIFIER)</string></array>  <!-- required with processing -->
```

plus iOS-only APIs `setBackgroundLocationAllowed()`,
`setBackgroundLocationIndicatorVisible()`, and pass `false` to
`setPauseLocationUpdatesAutomatically()` — all return
`LocationEngineStatus.notSupported` on Android; `notAllowed` if capabilities
missing.

- Track `AppLifecycleState` / `WidgetsBindingObserver` and stop the engine on
  dispose; avoid downloading in background (see `here-flutter-offline`).

## 6. Testing location

- iOS Simulator: **Features → Location → Custom Location**; Android Emulator:
  **Extended Controls → Location tab**.
- In-SDK simulation for navigation: `LocationSimulator.withRoute(route,
  LocationSimulatorOptions())` — see `here-flutter-navigation`.
- Recorded tracks: `hiking_diary_app` / GPX tutorial
  (`flutter-hiking-app-tutorial`) — includes the
  `DistanceAccuracyLocationFilter` pattern (accuracy ≤ 10 m, distance ≥ 15 m
  thresholds) to reject stationary jitter.

## Verify before delivering

- [ ] Navigate license confirmed; permissions in **both** manifest and
      Info.plist; runtime prompt via `permission_handler`
- [ ] `confirmHEREPrivacyNoticeInclusion()` called before `start()` on Android
- [ ] Exactly one started engine; listeners added before start and removed
      before stop; status listener handles all enum values
- [ ] `Location.time` set; accuracy level chosen for the use case; puck style
      matches transport mode
- [ ] Background: foreground service (Android) / UIBackgroundModes (iOS) —
      and only iOS-only APIs where they exist
- [ ] Privacy-policy text present (`shared/credentials-and-licensing.md` §6)
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
