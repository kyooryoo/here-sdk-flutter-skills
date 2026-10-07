---
name: here-flutter-routing
description: >
  Calculate and display routes with the HERE SDK for Flutter: waypoints,
  transport modes (car/truck/pedestrian/EV/bus/scooter), route options,
  avoid/allow features, tolls, alternatives, truck and EV routing, isolines,
  route refresh, return-to-route, import/serialize routes, maneuvers and ETA
  reading, and drawing the route on the map. Use for any routing task.
---

# HERE SDK Flutter — Routing

Prereqs: `here-flutter-bootstrap`. Online `RoutingEngine` = Explore + Navigate;
`OfflineRoutingEngine`, `IsolineRoutingEngine` details, waypoint optimization =
**Navigate only**. Callbacks: error/result XOR on the main thread.

## Calculate a route

```dart
try {
  _routingEngine = RoutingEngine();   // or OfflineRoutingEngine() (Navigate)
} on InstantiationException {
  throw Exception("Initialization of RoutingEngine failed.");
}

Waypoint startWaypoint = Waypoint.withDefaults(startGeoCoordinates);
Waypoint destinationWaypoint = Waypoint.withDefaults(destinationGeoCoordinates);
List<Waypoint> waypoints = [startWaypoint, destinationWaypoint];

_routingEngine.calculateRouteWithRoutingOptions(waypoints, RoutingOptions(),
    (RoutingError? routingError, List<HERE.Route>? routeList) async {
  if (routingError != null) { /* handle */ return; }
  // error == null ⇒ list guaranteed non-empty; 1 route unless alternatives requested
  HERE.Route route = routeList!.first;
  _showRouteOnMap(route);
});
```

- ⚠ Flutter's `Route`/`Navigator` clash with HERE's — always
  `import 'package:here_sdk/routing.dart' as HERE;` (or alias) and use
  `HERE.Route`.
- Multiple parallel calculations are allowed. Engines cannot be created in
  `Application.onCreate()`.
- Online limit ~**200 waypoints**; `OfflineRoutingEngine` has no limit
  (Navigate).

## Waypoints

- `Waypoint.withDefaults(coords)` / `Waypoint(coords)`; fields:
  `.coordinates`, `.sideOfStreetHint`, `.headingInDegrees` (set from
  `location.bearingInDegrees`), `.chargingStop`, type `WaypointType.stopover`
  (default) vs `passThrough`.
- First & last must be `stopover`; `stopover` splits the route into `Section`s
  and appears in maneuvers; `passThrough` is only a shaping hint.
- Place → waypoint: prefer `place.accessPoints.first` as coordinates with
  `sideOfStreetHint = place.geoCoordinates` (access points are
  vehicle-reachable).

```dart
Waypoint dest = Waypoint(place.geoCoordinates!);
if (place.accessPoints.isNotEmpty) {
  dest.coordinates = place.accessPoints.first;
  dest.sideOfStreetHint = place.geoCoordinates;
}
```

## Reading the route

```dart
int etaSeconds   = route.duration.inSeconds;        // includes traffic delay
int delaySeconds  = route.trafficDelay.inSeconds;
int lengthMeters  = route.lengthInMeters;
GeoPolyline geom  = route.geometry;                  // .vertices
GeoBox bounds     = route.boundingBox;
for (Section section in route.sections) {
  for (Maneuver m in section.maneuvers) {            // preview text from Route
    ManeuverAction action = m.action;  String text = m.text;  GeoCoordinates at = m.coordinates;
  }
  for (Span span in section.spans) { /* streetNames, truckAttributes, dynamicSpeedInfo,
    speedLimitInMetersPerSecond, consumptionInKilowattHours, noticeIndexes, ... */ }
}
```

- ⚠ Section durations are **cumulative** (last section holds totals).
- ⚠ `Maneuver.text` is non-empty **only when read from a `Route`** (preview);
  `roadTexts`/`nextRoadTexts`/`exitSignTexts` are non-empty **only during
  navigation** (read via `visualNavigator.getManeuver(i)`) — Navigate.
- **Always check violations:** iterate `span.noticeIndexes` →
  `section.sectionNotices[i].code` (e.g. `violatedVehicleRestriction`);
  empty list = clean route.
- ETA helpers: destination/device/UTC variants; arrival via
  `route.sections.last.arrivalLocationTime`.

## Route options (nested under `RoutingOptions`)

```dart
RoutingOptions routingOptions = RoutingOptions();
routingOptions.transportSpecification.transportMode = TransportMode.truck;   // car|truck|pedestrian|scooter|bicycle|taxi|bus|privateBus
routingOptions.routeOptions.optimizationMode = OptimizationMode.fastest;    // or .shortest
routingOptions.routeOptions.alternatives = 2;
routingOptions.routeOptions.enableRouteHandle = true;   // REQUIRED for refresh / traffic-on-route
routingOptions.routeOptions.enableTolls = true;         // → section.tolls
routingOptions.routeOptions.trafficOptimizationMode = TrafficOptimizationMode.timeDependent; // default; .disabled strips route traffic info + ETA delays
routingOptions.avoidanceOptions = AvoidanceOptions();   // RoadFeatures / ZoneCategory / GeoBox / CountryCode / .segments
routingOptions.allowOptions.allowHov = true;
```

- Defaults that bite: `enableRouteHandle = false` (blocks refresh/traffic),
  `enableTolls = false`, `alternatives = 0`, `trafficOptimizationMode =
  timeDependent` (leaving arrival/departure null = device time).
- Avoid segment: `SegmentReference()..segmentId = 'here:cm:segment:...'..
  tilePartitionId = ...` → `avoidanceOptions.segments = [ref]`. Confirmed road
  feature: `RoadFeatures.controlledAccessHighway` (tunnels/ferries/LEZ/dirty
  roads supported per prose — verify full enum via
  `here-flutter-docs-navigation`).
- HOV/HOT only with `vehicleSpecification.occupancy > 1` and country support.

## Truck & vehicle specification

```dart
VehicleSpecification spec = VehicleSpecification();
// dimensions/weights (height, length, width, grossWeightInKilograms, occupancy, ...)
routingOptions.transportSpecification.vehicleSpecification = spec;
```

- ⚠ Exact truck attribute names are documented in the API Reference — verify
  via `here-flutter-docs-navigation` before emitting them.
- `grossWeightInKilograms` also affects truck speed limits (CVR data: map
  version ≥ 32) and restrictions. Truck **offline** routing needs
  `LayerConfigurationFeature.truck` (default off).

## EV routing

```dart
ElectricVehicleOptions ev = ElectricVehicleOptions();
ev.empiricalConsumptionModel = EmpiricalConsumptionModel()
  ..ascentConsumptionInWattHoursPerMeter = 9
  ..descentRecoveryInWattHoursPerMeter = 4.3
  ..freeFlowSpeedTable = {0: 0.239, 27: 0.239, 60: 0.196, 90: 0.238};   // minimum required
ev.batterySpecifications = BatterySpecifications()
  ..connectorTypes = [ChargingConnectorType.tesla]   // + iec62196Type1Combo/iec62196Type2Combo
  ..totalCapacityInKilowattHours = 80.0
  ..initialChargeInKilowattHours = 10.0
  ..targetChargeInKilowattHours = 72.0
  ..chargingCurve = {0.0: 239.0, 64.0: 111.0, 72.0: 1.0};
ev.ensureReachability = true;
routingOptions.evOptions = ev;
```

- ⚠ `ensureReachability = true` **preconditions** (else error): alternatives
  = 0, empty `avoidanceOptions`, `speedCapInMetersPerSecond = null`,
  `optimizationMode = fastest`; battery trio
  (`totalCapacity`, `initialCharge`, `chargingCurve`) mandatory when charging
  stops are used → else `INVALID_PARAM`.
- ⚠ Billing: counted as "Routing EV" **only when `ensureReachability=true`**.
- Results: `sections.length - 1` = planned charging stops; read
  `section.consumptionInKilowattHours`, `postActions`
  (`PostActionType.charging` etc.), `departurePlace/arrivalPlace
  .chargeInKilowattHours` (negative arrival = unreachable),
  `routePlace.chargingStation`.
- User-planned stop: `Waypoint(coords)..chargingStop =
  ChargingStop(power, current, voltage, ChargingSupplyType.dc, min, max)`.
- EV routes **not supported offline** (`OfflineRoutingEngine` fails with
  `RoutingError.invalidParameter`).

## Draw the route

```dart
MapPolyline routeMapPolyline = MapPolyline.withRepresentation(
    route.geometry,
    MapPolylineSolidRepresentation(
        MapMeasureDependentRenderSize.withSingleSize(RenderSizeUnit.pixels, 20),
        const Color.fromARGB(160, 0, 144, 138), LineCap.round));
hereMapController.mapScene.addMapPolyline(routeMapPolyline);
// fit: camera.lookAtAreaWithGeoOrientation(route.boundingBox, GeoOrientationUpdate(null, null))
// animated fit: MapCameraUpdateFactory.lookAtAreaWithGeoOrientationAndViewRectangle(...)
```

Catch `MapPolylineRepresentationInstantiationException` /
`MapMeasureDependentRenderSizeInstantiationException`.

## Advanced

| Need | API (license) |
|---|---|
| Reach-area polygon | `IsolineRoutingEngine()` → `calculateIsoline(Waypoint, IsolineOptions.withRoutingOptions(IsolineOptionsCalculation.withNoDefaults(IsolineRangeType.consumptionInWattHours, [400], IsolineCalculationMode.balanced, maxPoints, RoutePlaceDirection.departure), evOptions), cb)` → `Isoline.polygons` → `MapPolygon` (Navigate; `alternatives` must be 0) — **verify range types** |
| Update ETA with traffic | `routeOptions.enableRouteHandle = true` → `calculateTrafficOnRoute(route, sectionIdx, dist, cb)` → `TrafficOnRoute` (billable) |
| Refresh after deviation/new start | `refreshRouteWithRouteHandleAndRefreshRouteParameters(RefreshRouteParameters.withRouteHandleAndWaypoint(handle, waypoint), opts, cb)` |
| Rejoin after leaving route | `routingEngine.returnToRouteWithTraveledDistance(originalRoute, newStart, lastSectionIdx, traveledDist, cb)` — recommended default; online needs `RouteHandle` (else `NO_ROUTE_HANDLE`) |
| Better route while driving | `DynamicRoutingEngine(options)` (Navigate, online, billable; needs route handle; call `updateCurrentLocation` or no events) |
| Persist / restore | `Route.serialize(...)` / `Route.deserialize(...)` |
| Import a trace | `importRouteWithRoutingOptions(List<Location>, opts, cb)` (dense points, online) or `importRouteFromHandleWithRoutingOptions(RouteHandle(str), opts, cb)` (handles valid only hours) |

- Offline engine routes have **empty maneuver lists** — take maneuvers from
  `Navigator`/`VisualNavigator` instead.

## Verify before delivering

- [ ] Error branch checked first; single-route assumption only when
      `alternatives == 0`
- [ ] `HERE.Route` aliasing avoids Flutter `Route` clash; `WaypointType`
      first/last stopover rules respected
- [ ] `enableRouteHandle` set **before** calculation if refresh/traffic-on-route
      will be needed
- [ ] Avoid/toll/HOV options deliberate; violations surfaced via
      `sectionNotices`
- [ ] EV preconditions met; truck attributes verified against API reference
- [ ] Maneuver source correct (Route preview vs navigator during guidance)
- [ ] Billable calls (`calculateTrafficOnRoute`, `DynamicRoutingEngine`,
      `ensureReachability`) called out to the user
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
