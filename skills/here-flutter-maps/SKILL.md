---
name: here-flutter-maps
description: >
  Work with the HERE SDK for Flutter map view: switch map schemes/themes
  (day-night, satellite, topo, logistics), animate the camera, handle gestures
  and taps, pick map items/POIs, add markers/polylines/polygons/circles/3D
  models/clusters/location puck, toggle map features (buildings, traffic,
  terrain), set map language and political view, apply custom styles/layers,
  and configure indoor maps. Use for any display, annotation, camera, or style
  task on a HERE Flutter map.
---

# HERE SDK Flutter — Maps, camera & map items

Prereqs: `here-flutter-bootstrap`. All SDK calls on the main isolate;
callbacks arrive on the main thread (`shared/threading-and-lifecycle.md`).

## Map scenes & schemes

```dart
hereMapController.mapScene.loadSceneForMapScheme(MapScheme.normalDay, (MapError? error) {
  if (error != null) { print('MapError: $error'); return; }
  // MapFeatures reset on every load — re-enable here
});
```

| Scheme constants | Notes |
|---|---|
| `normalDay` / `normalNight` | both licenses |
| `hybridDay` / `hybridNight`, `satellite`, `liteHybridDay/Night` | **raster-tile transactions** ($) |
| `liteDay` / `liteNight` | simplified, both licenses |
| `logisticsDay` / `logisticsNight` + hybrids | truck/fleet focus |
| `roadNetworkDay` / `roadNetworkNight` | roads only — good for nav mini-maps |
| `topoDay` / `topoNight` | **Navigate only** (explicit banner) |

- **The SDK does NOT auto-switch day/night** — "An application may decide to do
  this based on the user's configuration or by checking the device clock."
  Switching = call `loadSceneForMapScheme` again.
- Custom styles can replace everything **except satellite imagery**.
- Embedded (Carto) POIs show by default; hiding categories
  (`MapContentSettings.setPoiCategoriesVisibility`) is **Navigate only**.
- Projection: default globe; Mercator is Navigate-only per feature list.

Sources: `flutter-map-schemes`, `flutter-maps`

## Language, political view, watermark

```dart
hereMapController.primaryLanguage = LanguageCode.hiIn;   // null → local language
hereMapController.secondaryLanguage = LanguageCode.enUs;  // dual labels
SDKOptions.politicalView = "ARG";   // BETA; set BEFORE first init only
```

- Political view changes at runtime need a cache clear + dispose + re-init —
  "Ideally, set the options and the `politicalView` only once before
  initializing the HERE SDK for the first time."
- **Watermark:** HERE logo must remain visible; reposition via
  `setWatermarkLocation()` on the map view controller; auto-hidden when both
  edges < 250 dp; removal needs a HERE agreement.

## Camera (`hereMapController.camera`, class `MapCamera`)

| Need | Call |
|---|---|
| Jump to place | `camera.lookAtPointWithMeasure(GeoCoordinates(lat, lng), MapMeasure(MapMeasureKind.distanceInMeters, meters))` |
| Jump (no measure) | `camera.lookAtPoint(GeoCoordinates(...))` |
| Fit area | `camera.lookAtAreaWithGeoOrientation(geoBox, GeoOrientationUpdate(null, null))` (`GeoBox(topLeft, bottomRight)`) |
| With orientation | `camera.lookAtPointWithGeoOrientationAndMeasure(coords, GeoOrientationUpdate(bearing, tilt), mapMeasure)` |
| Zoom | `camera.zoomTo(level)` — levels **[0, 22]** (`Double`) |
| Read state | `camera.state.targetCoordinates`, `camera.boundingBox` |
| Shift center (HUD) | `camera.principalPoint = Point2D(w/2, h*0.75)` |
| Animate | `camera.startAnimation(MapCameraAnimationFactory.flyTo(GeoCoordinatesUpdate.fromGeoCoordinates(dest), bowFactor, Duration(seconds: 3)))` |
| Animated fit | `MapCameraUpdateFactory.lookAtAreaWithGeoOrientationAndViewRectangle(...)` + `MapCameraAnimationFactory.createAnimationFromUpdateWithEasing(update, dur, Easing(EasingFunction.inCubic))` |

Gotchas: `GeoOrientationUpdate(bearing, tilt)` fields are **nullable** (null =
keep); bearing default 0 (north-up, clockwise); **tilt is absolute, not a
delta**; out-of-range values silently clamped; don't mix `distanceInMeters`
with zoom level (both `Double`). Camera limits:
`MapCameraLimits.minTilt` / `.maxZoomLevel`. For navigation-owned camera use
`FixedCameraBehavior().normalizedPrincipalPoint` on `VisualNavigator`
(Navigate) instead of hacking the map camera.

## Gestures & taps

```dart
hereMapController.gestures.tapListener = TapListener((Point2D touchPoint) {
  GeoCoordinates? geo = hereMapController.viewToGeoCoordinates(touchPoint); // nullable!
});
hereMapController.gestures.tapListener = null;   // detach
```

- Listener classes: `TapListener`, `DoubleTapListener`, `LongPressListener`,
  `PanListener`, `TwoFingerPanListener`, `TwoFingerTapListener`,
  `PinchRotateListener` — **one listener per gesture**.
- Continuous gestures pass `GestureState.begin/update/end/cancel`.
- Defaults: double tap = zoom in, two-finger tap = zoom out, pan = move,
  two-finger pan = tilt, pinch = zoom+rotate. Toggle independent of listeners:
  `gestures.disableDefaultAction(GestureType.doubleTap)` /
  `enableDefaultAction(...)`.
- `viewToGeoCoordinates` returns **null** when the touch point maps above the
  horizon (tilted view) — always null-check.
- Idle/end-of-fling detection: `MapIdleListener`.

## Picking (inside/after scene load)

```dart
List<MapSceneMapPickFilterContentType> types = [MapSceneMapPickFilterContentType.mapItems];
// or .mapContent → embedded POIs, traffic incidents, vehicle restrictions
MapSceneMapPickFilter filter = MapSceneMapPickFilter(types);
hereMapController.pick(filter, Rectangle2D(touchPoint, Size2D(50, 50)), (pickResult) {
  PickMapItemsResult? items = pickResult?.mapItems;
  MapMarker topmost = items!.markers!.first;
  PickMapContentResult? content = pickResult?.mapContent;
  List<PickedPlace> places = content!.pickedPlaces!;
});
```

- Pickable: `MapMarker`, `MapPolyline`, `MapPolygon`, `MapArrow`, cluster
  groups, embedded POIs, traffic incidents (via `pickMapContent`).
- **Not pickable:** `MapMarker3D`, `LocationIndicator`, markers inside a
  cluster (use `clusteredMarkers` → `MapMarkerClusterGrouping`), pins (use the
  widget's own `GestureDetector`).
- Traffic incidents are "not directly pickable yet" via item filter — pick via
  `hereMapController.pickMapContent(rectangle, cb)` → `.trafficIncidents`.

## Map items (add to `hereMapController.mapScene`)

```dart
// Marker
MapImage mapImage = MapImage.withFilePathAndWidthAndHeight("assets/pin.png", 60, 60); // size REQUIRED
// or async: MapImage.withPixelDataAndImageFormat(bytes, ImageFormat.png)
MapMarker marker = MapMarker(geoCoordinates, mapImage);
// MapMarker.withAnchor(coords, image, Anchor2D.withHorizontalAndVertical(0.5, 1))
marker.text = "Hello"; marker.isOverlapAllowed = false; marker.drawOrder = 2;
mapScene.addMapMarker(marker);           // batch: addMapMarkers(list)

// Polyline (≥ 2 points)
GeoPolyline geoPolyline = GeoPolyline(vertices);   // throws if <2
MapPolyline line = MapPolyline.withRepresentation(geoPolyline,
    MapPolylineSolidRepresentation(
        MapMeasureDependentRenderSize.withSingleSize(RenderSizeUnit.pixels, 8),
        color, LineCap.round));
mapScene.addMapPolyline(line);

// Polygon / circle (circle = polygon over GeoCircle)
MapPolygon poly = MapPolygon(GeoPolygon(vertices), fillColor);           // ≥3 points
MapPolygon circle = MapPolygon(GeoPolygon.withGeoCircle(GeoCircle(center, 500.0)), color);
```

| Want | API |
|---|---|
| Arrow along a path | `MapArrow(geoPolyline, widthPx, color)` — visible above zoom 13 |
| Cluster markers | `MapMarkerCluster.WithCounter(MapMarkerClusterImageStyle(image), counterStyle)` + `c.addMapMarker(m)` |
| 3D model | `MapMarker3DModel.withTextureFilePath(obj, png)` + `MapMarker3D(coords, model)`; `m.isDepthCheckEnabled = true` to allow occlusion; add via `addMapMarker3d(...)` (lowercase d!) |
| Flat marker | `MapMarker3D.fromImage(coords, image, scale, RenderSizeUnit.densityIndependentPixels)` |
| Puck / current location | `LocationIndicator` — `enable(controller)`, `updateLocation(location)`, `locationIndicatorStyle = LocationIndicatorIndicatorStyle.navigation/.pedestrian`, `isAccuracyVisualized = true` |
| Attach data | `Metadata()..setString("k", "v")` → `mapMarker.metadata`; custom objects via `CustomMetadataValue` |
| Gradient/progress line | `mapPolyline.progressColor`, `.progress = 0.4`, `.progressGradientLength` |
| Zoom-limited visibility | `mapPolyline.visibilityRanges = [MapMeasureRange(MapMeasureKind.zoomLevel, 1, 10)]` (half-open) |
| Custom label fonts | `AssetsManager(controller.mapContext).registerFont("SimpleSans", "assets/fonts/X.ttf")` |
| Move marker | `mapMarker.setCoordinates(...)`; animate with `MapMarkerAnimation` |
| Native widget on map | `controller.pinWidget(widget, geo)` → `WidgetPin` (lags during pan; prefer markers for many) |

Rules: polylines/polygons/circles scale with zoom; markers/pins are
screen-fixed; **drawOrder only orders within the same item class** — markers
always render on top; coordinate order in a polygon matters (self-intersections
render badly); Flutter's `2.0x/3.0x` asset folders are **not** auto-resolved by
the SDK — specify path+size per density; PNG and SVG Tiny supported.

## Map features (reset on every scene load)

```dart
hereMapController.mapScene.enableFeatures({
  MapFeatures.extrudedBuildings: MapFeatureModes.extrudedBuildingsAll,
  MapFeatures.trafficFlow: MapFeatureModes.trafficFlowWithFreeFlow,
});
hereMapController.mapScene.disableFeatures([MapFeatures.terrain]);
```

Key constants (+ license): `buildingFootprints`*, `extrudedBuildings`*,
`trafficFlow`, `trafficIncidents`, `landmarks`**N**, `terrain`**N**
(`terrain3d` = beta), `contours`**N**, `publicTransit`**N**,
`safetyCameras`**N**, `vehicleRestrictions`**N**, `lowSpeedZones`,
`environmentalZones`, `congestionZones`, `trafficLights`, `roadExitLabels`,
`shadows`, `ambientOcclusion`. (* = on by default.)

- ⚠ "all previously configured map features are reset to their default state.
  You must re-enable your desired map features after each scene load
  operation." Changing features *while* a scene loads may throw.
- ⚠ `$` cost: `trafficFlow`/`trafficIncidents` each trigger vector-tile
  requests per new tile → consider `TrafficOnRoute` instead during navigation;
  refresh period via `MapContentSettings.setTrafficRefreshPeriod` (1–5 min).
- ⚠ GPU restrictions on some devices (PowerVR Rogue, Adreno 730–740, Mali
  32–38): `ambientOcclusion`/`shadows` prohibited, zone layers darken
  junctions, transparent polylines lose alpha — see changelog known issues.
- `shadows` needs `hereMapController.shadowQuality = ShadowQuality.veryHigh`
  to take effect.

## Custom styles & layers

- Style Editor: <https://platform.here.com/style-editor/> → **File → Export
  Map Style**. Since v1.13.0 exports `.tar.gz` → **extract and re-zip as
  `.zip`** into `assets/` (declared in `pubspec.yaml`); older exports are raw
  `.json`.

  ```dart
  hereMapController.mapScene.loadSceneFromConfigurationFile("assets/my-style.zip", (MapError? e) {...});
  // after editing files on disk: mapScene.reloadScene()  (bypasses stale cache)
  ```

- Custom layers (raster/line/polygon/point/tile) are **beta — "APIs may change
  for new releases without a deprecation process"**:
  `RasterDataSource(mapContext, RasterDataSourceConfiguration.withDefaults(...))`
  → `MapLayerBuilder().forMap(controller.hereMapControllerCore)
  .withName(name).withDataSource(ds, MapContentType.rasterImage)
  .withPriority(MapLayerPriorityBuilder().renderedAfterLayer("labels").build())
  .withVisibilityRange(MapLayerVisibilityRange(MapCameraLimits.minTilt,
  MapCameraLimits.maxZoomLevel)).withStyle(style).build()` →
  `mapLayer.setEnabled(true)`.
  - Style JSON `"layer"` value must match `withName(...)`; tile URL template
    `{z}/{x}/{y}` via `TileUrlProviderFactory.fromXyzUrlTemplate`.
  - Runtime style edits: `JsonStyleFactory.createFromString(json)` (**beta**).
  - Custom road geometry/POIs require HERE support + agreement.

## Indoor maps (Navigate only, beta banner)

- `VenueEngineWidget(state: _venueEngineState)` stacked over `HereMap`;
  `VenueEngine(cb)` → `venueEngine.venueService.loadTopologies()` →
  `venueEngine.start(callback)`; disable `MapFeatures.extrudedBuildings`
  while showing a venue; venues are private, listed via
  `venueMap.getVenueInfoList()`, shown with
  `venueMap.selectVenueAsync(venueIdentifier)`.
- Indoor routing: `IndoorRoutingEngine(venueEngine.venueService)` →
  `IndoorWaypoint(...)` → `calculateRouteWithRouteNotices(...)` →
  `IndoorRoutingController.showRoute(route, IndoorRouteStyle())`. **Check
  `routeNotices` even on success** (4.28 moved errors into notices).
- Full guide: `flutter-venues`, `flutter-venues-use-cases`.

## Verify before delivering

- [ ] Scene loaded via callback with error branch; `MapFeatures` re-enabled
      after **every** `loadSceneForMapScheme`
- [ ] Day/night switching is app-driven where required
- [ ] All camera/gesture/map-item calls on the main isolate
- [ ] Pick logic only uses pickable types; null-checks on
      `viewToGeoCoordinates` and pick results
- [ ] Marker icons sized explicitly (`withFilePathAndWidthAndHeight`);
      polygons have ≥3 vertices; exceptions caught
- [ ] HERE watermark visible; satellite schemes & traffic features justified
      (transactions/$)
- [ ] Beta flags called out (custom layers, style runtime edit, political
      view, terrain3d, indoor maps)
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
