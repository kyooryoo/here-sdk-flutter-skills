---
name: here-flutter-traffic
description: >
  Show and query traffic with the HERE SDK for Flutter: traffic flow and
  incident map layers, refresh periods and cost control, querying incidents
  and flow with TrafficEngine, picking incidents, traffic on a route
  (TrafficOnRoute / jam factors), and choosing between refresh, traffic-on-route
  and DynamicRoutingEngine. Use for any traffic visualization or traffic-aware
  ETA task.
---

# HERE SDK Flutter — Traffic

Prereqs: `here-flutter-bootstrap`, `here-flutter-maps`. All traffic features
are available under **both** licenses; rendering+querying counts **transactions
($)** — see cost rules below. `VisualNavigator.trafficOnRoute` rendering is
Navigate-only.

## 1. Map layers (visualize)

```dart
try {
  MapContentSettings.setTrafficRefreshPeriod(Duration(minutes: 5));   // legal range 1–5 min
} on MapContentSettingsTrafficRefreshPeriodExceptionException {       // sic, double "Exception"
  throw ("TrafficRefreshPeriodException");
}
hereMapController.mapScene.enableFeatures({
  MapFeatures.trafficFlow: MapFeatureModes.trafficFlowWithFreeFlow,
  MapFeatures.trafficIncidents: MapFeatureModes.defaultMode,
});
// off:
hereMapController.mapScene.disableFeatures([MapFeatures.trafficFlow, MapFeatures.trafficIncidents]);
```

- ⚠ **Scene-load reset:** re-enable after every `loadSceneForMapScheme`.
- Flow colors: Green = normal, Amber = high, Red = very high, Black = blocking.
- Predicted vs real-time: flow layer ≈ real-time (Traffic Vector Tile API);
  route/`TrafficOnRoute` ≈ predictive (Traffic API v7) — discrepancies are
  expected.
- Other flow modes: `trafficFlowWithoutFreeFlow`,
  `trafficFlowJapanWithoutFreeFlow`, `defaultMode`.

## 2. Cost rules ($)

> "each feature request is counted as an individual transaction… flow and
> incidents enabled = two separate transactions… requests are triggered when
> new vector tiles are loaded during panning/zooming or when the validity time
> expires while tiles are visible."

- Refresh: `MapContentSettings.setTrafficRefreshPeriod` (1–5 min).
- ⚠ During turn-by-turn navigation "viewport changes can happen multiple times
  per second and may result in a high number of Traffic Vector Tile requests…
  consider using TrafficOnRoute instead."
- `MapContentSettings` can also filter incidents:
  `filterTrafficIncidents(List<TrafficIncidentType>)`.

## 3. Query incidents (`TrafficEngine`)

```dart
try {
  _trafficEngine = TrafficEngine();
} on InstantiationException {
  throw Exception("Initialization of TrafficEngine failed.");
}

GeoCircle geoCircle = GeoCircle(centerCoords, 1000);   // also GeoBox, GeoCorridor
TrafficIncidentsQueryOptions queryOptions = TrafficIncidentsQueryOptions();
// queryOptions.languageCode = LanguageCode.enUs;   // default = incident country language
_trafficEngine.queryForIncidentsInCircle(geoCircle, queryOptions,
    (TrafficQueryError? error, List<TrafficIncident>? incidents) {
  if (error != null) return;                    // XOR: error==null ⇒ list non-empty
});

_trafficEngine.lookupIncident(originalId, TrafficIncidentLookupOptions(),
    (TrafficQueryError? e, TrafficIncident? incident) { ... });
```

- **Error enum is `TrafficQueryError`** (not `SearchError`).
- Incident results are **not sorted by distance** — compute nearest yourself
  (`geoCoords.distanceTo`) over `incident.location.polyline`.
- `TrafficIncident`: `.description.text`, `.location.polyline` (always valid),
  `.location.additionalPolylines` (gaps), `.vehicleRestrictions` (map keyed by
  `TrafficIncidentRestrictedVehicleCategory`: `isRestrictedAlways`,
  `restrictedIfGrossWeightMoreThanInKilograms`, …).

## 4. Query flow

```dart
GeoCorridor corridor = GeoCorridor(route.geometry.vertices, 500);   // halfWidth in meters
TrafficFlowQueryOptions flowOptions = TrafficFlowQueryOptions();
_trafficEngine.queryForFlowInCorridor(corridor, flowOptions,
    (TrafficQueryError? e, List<TrafficFlow>? flows) {
  for (TrafficFlow f in flows!) {
    double? jam = f.jamFactor;             // 0..10
    double? confidence = f.confidence;      // <= 0.5 → speed-limit-derived, consider skipping
    // f.location.polyline → draw MapPolyline colored by jam factor
  }
});
```

- Jam-factor bands (routing context): `<4` none/light · `4–<8` moderate ·
  `8–<10` severe · `10` blocked.
- Exact box-based flow/incident method names not shown in the guide — verify
  via `here-flutter-docs-navigation`.

## 5. Pick incidents on the map

```dart
hereMapController.pickMapContent(Rectangle2D(touchPoint, Size2D(50, 50)), (pickMapContentResult) {
  if (pickMapContentResult == null) return;
  List<PickTrafficIncidentResult> incidents = pickMapContentResult.trafficIncidents;
  if (incidents.isEmpty) return;
  String originalId = incidents.first.originalId;
  // then trafficEngine.lookupIncident(originalId, ...) for full details
});
```

- Incidents rendered as a map feature are "not directly pickable yet" via the
  item filter — use `pickMapContent`.

## 6. Traffic on a route (predictive ETA)

```dart
// Requires routeOptions.enableRouteHandle = true at calculation time
_routingEngine.calculateTrafficOnRoute(
    route, lastTraveledSectionIndex /* 0 in preview */, traveledDistanceOnLastSectionInMeters /* 0 */,
    (RoutingError? routingError, TrafficOnRoute? trafficOnRoute) { ... });

for (var section in trafficOnRoute!.trafficSections) {
  int eta = section.trafficSpans.fold(0, (sum, s) => sum + s.duration.inSeconds);
  int delay = section.trafficSpans.fold(0, (sum, s) => sum + s.trafficDelay.inSeconds);
}
```

- During guidance: `_visualNavigator.trafficOnRoute = trafficOnRoute;` —
  reflected in the **next** `RouteProgress`, not immediately. Recommended
  interval 5–15 min (billable backend calls).
- Per-span rendering: `span.dynamicSpeedInfo` →
  `baseSpeedInMetersPerSecond`, `trafficSpeedInMetersPerSecond`,
  `calculateJamFactor()` (from speed ratio only — may not exactly match the
  map layer). For long routes render selectively (one polyline per span is
  expensive; docs sample skips routes > 5000 km).

## 7. Decision matrix (verbatim from `flutter-traffic-features`)

| Need | Use |
|---|---|
| Refresh preview-route ETA | `routingEngine.calculateTrafficOnRoute(..)` |
| Update an existing route instance (same shape) | `refreshRouteWithRouteHandleAndRefreshRouteParameters()` |
| Turn-by-turn ETA + rendering | `calculateTrafficOnRoute()` + `VisualNavigator.trafficOnRoute` (Navigate) |
| Find a better route while driving | `DynamicRoutingEngine` (Navigate, billable) |
| Show live flow next to route | adjacent polylines / `DynamicSpeedInfo` overlay; Navigate default widths via `VisualNavigator.defaultRouteManeuverArrowMeasureDependentWidths()` |

- Radio RDS-TMC traffic (`TrafficBroadcast`, offline) = **Navigate + special
  hardware + beta**.

## Verify before delivering

- [ ] Traffic features re-enabled after every scene load; refresh period set
      (1–5 min) with the double-Exception type caught
- [ ] Transaction cost of layers/queries explained; navigation viewport spike
      avoided via `TrafficOnRoute`
- [ ] `enableRouteHandle` set before any `calculateTrafficOnRoute` call
- [ ] XOR callbacks checked; incidents manually distance-sorted if needed
- [ ] Predictive vs real-time discrepancy not presented as a bug
- [ ] Beta/Navigate-only items flagged (radio broadcast, navigator rendering)
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
