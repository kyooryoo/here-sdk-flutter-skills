---
name: here-flutter-navigation
description: >
  Turn-by-turn navigation with the HERE SDK for Flutter: VisualNavigator vs
  Navigator, starting/stopping guidance, route progress and maneuvers, voice
  guidance/TTS and natural guidance, deviation and rerouting (returnToRoute,
  DynamicRoutingEngine), warners (WarnerEngine, speed/safety camera/truck),
  lane guidance, Electronic Horizon, truck navigation, camera behaviors, and
  LocationSimulator for testing. Use for any navigation/guidance task.
---

# HERE SDK Flutter — Navigation

Prereqs: `here-flutter-bootstrap`, `here-flutter-positioning`,
`here-flutter-routing`. **Navigate license only** — pages carry *"Navigation
is only available with the Navigate license."*
`import 'package:here_sdk/routing.dart' as HERE;` to dodge Flutter's
`Route`/`Navigator` clash.

## 1. Core concepts

- **Guidance (TBT):** needs a `Route`; produces progress, maneuvers, deviation
  events. **Tracking mode:** `visualNavigator.route = null` — map-matched
  position + warners, no `RouteProgressListener` events. Not supported for
  `PUBLIC_TRANSIT`.
- **`VisualNavigator`** = `Navigator` (headless logic) + rendering/interpolation
  on the `HereMapController`. Both implement `LocationListener`; any location
  source works as long as it calls `onLocationUpdated`.
- Guidance starts automatically when **`route` is set AND the location source
  is running**.

## 2. Start guidance (canonical)

```dart
import 'package:here_sdk/navigation.dart';
import 'package:here_sdk/routing.dart' as HERE;

void _startGuidance(HERE.Route route) {
  try {
    _visualNavigator = HERE.VisualNavigator();
  } on InstantiationException {
    throw Exception("Initialization of VisualNavigator failed.");
  }
  _visualNavigator!.startRendering(hereMapController);   // smooth interpolation + 3D arrow

  _visualNavigator!.routeProgressListener = RouteProgressListener((routeProgress) {
    updateProgress(routeProgress);
  });
  _visualNavigator!.eventTextListener = HERE.EventTextListener((HERE.EventText eventText) {
    speakOrDisplay(eventText.text);                      // feed to a TTS plugin
  });

  _visualNavigator!.route = route;
  _positioningProvider.startLocating(_visualNavigator!, LocationAccuracy.navigation);
}
```

- Stop: set `route = null` (→ tracking) or `stopRendering()` **before
  destroying the map**; null out listeners individually.
- Feed `LocationAccuracy.navigation` (≈1 Hz); `Location.time` mandatory.
- ⚠ During guidance take `Maneuver` via `_visualNavigator.getManeuver(index)` —
  **never from the `Route`** (`roadTexts`/`nextRoadTexts`/`exitSignTexts` are
  empty on route-sourced maneuvers, non-empty only during navigation).

## 3. Progress & events

```dart
_visualNavigator.routeProgressListener = RouteProgressListener((RouteProgress rp) {
  List<SectionProgress> sections = rp.sectionProgress;      // non-empty
  SectionProgress last = sections.last;
  last.remainingDistanceInMeters;  last.trafficDelay.inSeconds;   // per-section (accumulated)
  int nextIdx = rp.maneuverProgress.first.maneuverIndex;
  Maneuver? m = _visualNavigator!.getManeuver(nextIdx);
  ManeuverAction action = m!.action;   m.text;   m.coordinates;
});
```

Other listeners (settable properties): `DestinationReachedListener`,
`MilestoneStatusListener` (waypoint reached/missed — `passThrough` excluded by
default), `NavigableLocationListener`, `RoadTextsListener`,
`OffRoadProgressListener`, `OffRoadDestinationReachedListener`,
`InterpolatedLocationListener`, `RouteDeviationListener`.
`navigator.calculateRemainingDistanceInMeters(coords)` → `double?`.

- ETA refresh during guidance (billable, 5–15 min interval):

  ```dart
  _routeCalculator.calculateTrafficOnRoute(route, lastSectionIdx, traveledDist,
      (RoutingError? e, TrafficOnRoute? t) { _visualNavigator!.trafficOnRoute = t; });
  ```

  Reflected in the **next** progress event, not immediately.

## 4. Voice guidance (⚠ NO audio included)

> "the HERE SDK does not include pre-recorded voice skins. This means, you need
> to integrate a TTS engine for playback."

- Voice "skins" in `voice_assets` are **text/localization data only** — use a
  Flutter TTS plugin (e.g. `flutter_tts`) with `EventText.text`.
- Options **before a route is set**:

  ```dart
  ManeuverNotificationOptions opts = ManeuverNotificationOptions.withDefaults();
  opts.language = LanguageCode.enUs;                 // 37 languages:
  // VisualNavigator.getAvailableLanguagesForManeuverNotifications()
  opts.unitSystem = UnitSystem.metric;
  opts.includedNaturalGuidanceTypes = [NaturalGuidanceType.trafficLight]; // off by default
  opts.includedNotificationTypes = [ManeuverNotificationType.range,
      ManeuverNotificationType.reminder, ManeuverNotificationType.distance,
      ManeuverNotificationType.action];   // omitting a type disables it entirely
  opts.enablePhoneme = true;   opts.notificationFormatOption = NotificationFormatOption.ssml;
  _visualNavigator!.maneuverNotificationOptions = opts;
  ```

- Timing per mode/road type:
  `getManeuverNotificationTimingOptionsWithTimingProfile(TransportMode.car,
  TimingProfile.fastSpeed)` → set `distanceNotificationDistanceInMeters` (default
  car/highway 3rd notification = **1300 m**) → `setManeuverNotificationTimingOptionsWithTimingProfile(...)`.
- Spatial audio: `EventTextOptions.enableSpatialAudio` →
  `SpatialNotificationDetails` → `SpatialTrajectoryData` (example:
  `spatial_audio_navigation_app`).
- Safety-camera TTS needs `SafetyCameraWarningOptions.enableTextNotification = true`
  + a `SafetyCameraWarningListener`.

## 5. Deviation & rerouting (SDK NEVER auto-reroutes)

```dart
_visualNavigator!.routeDeviationListener = RouteDeviationListener((RouteDeviation rd) {
  // rd.currentLocation.mapMatchedLocation (may be null) / .originalLocation
  // rd.lastLocationOnRoute  ← NULLABLE (never followed route)
  // rd.lastTraveledSectionIndex, rd.traveledDistanceOnLastSectionInMeters
});
```

- ⚠ Fires **per location update** while off-route → debounce (a few events /
  seconds); never start a route calculation on every event.
- Reactions (docs' order): 1) full recalculation with `RoutingEngine`
  (set `headingInDegrees` on first waypoint), 2) **`returnToRouteWithTraveledDistance`**
  — recommended default, 3) refresh with route handle (no path back from
  deviation — insufficient alone), 4) `DynamicRoutingEngine`.

  ```dart
  routingEngine.returnToRouteWithTraveledDistance(
      originalRoute, newStartingPoint, rd.lastTraveledSectionIndex,
      rd.traveledDistanceOnLastSectionInMeters,
      (RoutingError? e, List<HERE.Route>? routes) { ... });
  // online original route MUST have a RouteHandle (enableRouteHandle=true) else NO_ROUTE_HANDLE
  ```

- Better-while-driving: `DynamicRoutingEngine(DynamicRoutingEngineOptions()
  ..minTimeDifference = Duration(seconds: 1) ..pollInterval = ...)`
  → `start(route, DynamicRoutingListener(onBetterRouteFound, onError))` +
  **must** call `updateCurrentLocation(...)`; needs route handle; Navigate +
  online + billable; when swapping routes do `stop()` then `start()` **outside**
  `onBetterRouteFound` (flag + next location update).

## 6. Warners (`WarnerEngine` — unified since 4.26)

```dart
WarnerEngine warnerEngine = _visualNavigator!.warnerEngine;
WarningOptions warningOptions = warnerEngine.warningOptions;
warningOptions.safetyCameraWarningOptions = SafetyCameraWarningOptions()..enableTextNotification = true;
warnerEngine.warningOptions = warningOptions;

warnerEngine.setEnabledWarnings([WarningType.safetyCamera, WarningType.truckRestriction, ...]); // REQUIRED!
warnerEngine.addWarningListener(WarningListener((List<WarningUpdate> warnings, WarningsRegistry registry) {
  for (WarningUpdate w in warnings) {
    if (w.warning.warningType == WarningType.safetyCamera) {
      SafetyCameraWarning? sw = registry.getSafetyCameraWarning(w.warning);
      // w.warningStatus: ahead | approaching | reached | passed ; w.distanceTillStartInMeters
    }
  }
}));
// on stop: warnerEngine.finalizeGivenWarnings();
```

- ⚠ **Without `setEnabledWarnings([...])` no warnings are ever delivered** —
  listeners alone are not enough.
- ⚠ **Overspeed stays on the navigator**, not `WarnerEngine`:

  ```dart
  _visualNavigator!.speedWarningListener = SpeedWarningListener((SpeedWarningStatus s) {
    if (s == SpeedWarningStatus.speedLimitExceeded) { /* warn */ }
  });
  _visualNavigator!.speedLimitListener = SpeedLimitListener((SpeedLimit l) {
    l.effectiveSpeedLimitInMetersPerSecond();
  });
  _visualNavigator!.speedWarningOptions =
      SpeedWarningOptions(SpeedLimitOffset()..lowSpeedOffsetInMetersPerSecond = 2
                                            ..highSpeedOffsetInMetersPerSecond = 4);
  ```

- Works with or without a route (tracking). Registry getters:
  `getRoadSignWarning`, `getTollStopWarning`, `getSchoolZoneWarnings`,
  `getBorderCrossingWarning`, `getDangerZoneWarning`, `getLowSpeedZoneWarning`,
  `getTrafficMergeWarning`, `getEnvironmentalZoneWarnings`,
  `getRealisticViewWarning`, `getTruckRestrictionWarning`, `getCustomWarning`.
- Notification distances: `getWarningNotificationDistances(WarningType.roadSign)`
  → `fast/regular/slowSpeedDistanceInMeters` (defaults vary by section — verify
  against API ref). Not applicable to the speed (overspeed) warner.
- Custom warners: `CustomWarningProvider(...)` +
  `warnerEngine.addCustomWarningProvider(provider, segmentDataLoaderOptions)`.
- ⚠ Safety cameras excluded by law in many countries (long list incl. France,
  Germany). Road signs: events twice (`ahead` then `passed`); no icon/location
  data; speed-limit signs excluded.

## 7. Lane guidance (needs a route)

```dart
_visualNavigator!.maneuverViewLaneAssistanceListener =
    ManeuverViewLaneAssistanceListener((ManeuverViewLaneAssistance a) {
  List<Lane> lanes = a.lanesForNextManeuver;        // guaranteed non-empty
  // a.lanesForNextNextManeuver — usually empty
});
// JunctionViewLaneAssistanceListener — empty list = junction passed
// CurrentSituationLaneAssistanceViewListener — works in tracking too
```

- `Lane`: `.directions`, `.directionsOnRoute`,
  `.recommendationState` (`recommended|highlyRecommended|notRecommended`),
  `.directionCategory`, `.type`, `.access`, `.laneMarkings`.
- ⚠ Lanes are **always ordered leftmost (index 0) → rightmost** regardless of
  driving side; check `roadAttributes.isRightDrivingSide` for country side.
- Fires ~300 m ahead (1300 m highway). No TTS lane text by default — enable
  `maneuverNotificationOptions.enableLaneRecommendation` (18 s threshold,
  `distance` type only). Icons: HERE Icon Library.

## 8. Electronic Horizon (Navigate, **beta**)

- `ElectronicHorizonEngine.WithOptionsAndRoutePathEvaluator(engine,
  ElectronicHorizonOptions([1000.0, 500.0, 250.0], trailingMeters), transportMode, route)`
  → feed **map-matched** locations only:
  `_electronicHorizonEngine.update(mapMatchedLocation);`
- `ElectronicHorizonListener` → `ElectronicHorizonDataLoader.loadData(update)` →
  status listener → `getSegment(directedOCMSegmentId.id)` → `SegmentData`
  spans (street names, speed limits, road signs).
- "Related APIs may change for new releases without a deprecation process."
  No ADASISv2/v3.

## 9. Truck navigation

- Route with truck `VehicleSpecification` (`here-flutter-routing`); warners:
  `setEnabledWarnings([WarningType.truckRestriction])` →
  `registry.getTruckRestrictionWarning(...)` → `.weightRestriction`
  (`WeightRestrictionType`, `.valueInKilograms`), `.dimensionRestriction`
  (`.valueInCentimeters`), `.timeRule?.appliesTo(now)`.
- ⚠ Warnings arrive regardless of the configured transport mode → **filter
  irrelevant restrictions app-side**. Setting `route = null`/new route flips
  all pending `ahead` warnings to `passed`; deviating >15 m re-enables on the
  current road.
- Tracking mode assumes **car** speed limits unless the truck profile is set.
- Layers: `MapFeatures.safetyCameras`, `MapFeatures.vehicleRestrictions`.
- Truck CVR speed limits need map version ≥ 32 (`MapUpdater`).

## 10. Rendering & camera

- `startRendering()` = smooth interpolation + auto map orientation/tilt;
  guidance frame rate drops 60 → 30 fps (override:
  `visualNavigator.guidanceFrameRate = 60`).
- Camera behaviors: `visualNavigator.cameraBehavior = FixedCameraBehavior()`
  / `DynamicCameraBehavior()` / `SpeedBasedCameraBehavior()` (best for
  tracking) / `null` (stop following). Use
  `FixedCameraBehavior().normalizedPrincipalPoint` for HUD-style centering.
- Tunnel extrapolation on by default; in tunnels `NavigableLocation`'s
  estimated `Location` has `time`/`speed` intentionally null.
- Route eat-up styling: `isRouteProgressVisible`, `VisualNavigatorColors.dayColors()`,
  `setRouteProgressColors(...)`.
- Off-road destination: `isOffRoadDestinationVisible` (default on);
  `OffRoadProgressListener` gives remaining distance + bearing; once off-road
  events start there is no return without a new route.
- Custom puck: set a `LocationIndicator` on the navigator (you then own
  add/remove/update).

## 11. Testing with `LocationSimulator`

```dart
_locationSimulator = HERE.LocationSimulator.withRoute(route, HERE.LocationSimulatorOptions()
    ..speedFactor = 3.0            // default 1.0 = segment average; inverse relation to interval
    ..notificationInterval = Duration(seconds: 1));
_locationSimulator!.listener = locationListener;
_locationSimulator!.start();
```

- ⚠ `VisualNavigator` **skips animations if consecutive locations are >100 m
  apart** — keep speedFactor sane. Simulator locations are raw; interpolation
  happens in `VisualNavigator`. Example: `navigation_quick_start_app`.

## Verify before delivering

- [ ] Navigate license confirmed; `HERE` import alias avoids Flutter clashes
- [ ] Sequence: create navigator → `startRendering` → listeners → `route` →
      location source; `Location.time` set; accuracy `.navigation`
- [ ] Maneuvers read from the navigator during guidance, from `Route` only in
      preview
- [ ] `stopRendering()` + listener cleanup on teardown; `route = null` only
      when tracking is intended
- [ ] Deviation handler debounced; `returnToRoute` needs route handle
- [ ] `setEnabledWarnings([...])` called; overspeed listeners on the navigator;
      `finalizeGivenWarnings()` on stop
- [ ] Voice: TTS plugin wired; options set **before** route; notification
      types explicitly chosen
- [ ] Beta called out (Electronic Horizon, MapMatcher, RealisticView,
      lane-decrease); billable calls listed (`calculateTrafficOnRoute`,
      `DynamicRoutingEngine` polling)
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
