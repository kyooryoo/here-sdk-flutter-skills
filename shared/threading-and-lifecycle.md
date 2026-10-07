# HERE SDK for Flutter — Threading, callbacks & lifecycle (shared core)

The rules below cause most generated-code bugs. They apply to **every**
`here-flutter-*` skill.

## 1. Threading rules (verbatim-derived)

> "The HERE SDK is not guaranteed to be thread safe and it is required to make
> calls to the SDK from the main thread. Internally, the HERE SDK will offload
> most of its work to a background thread, but callbacks to your code will
> always occur on the main thread. In general, thread safety is the
> responsibility of the caller." — `flutter-tips-supplements`

- **Call all SDK APIs from the main isolate/thread.** Do not touch engines,
  `HereMapController`, or map items from a custom Dart isolate.
- **All callbacks/listeners arrive on the main thread** — safe to call
  `setState` directly.
- `SdkContext.init(IsolateOrigin.main)` is per-isolate: a *custom* isolate
  would need its own init with a different parameter — in practice treat the
  SDK as **main-isolate-only** unless proven otherwise. Hot restart destroys
  isolates → init runs again.
- **Exception (background-thread-safe, single-thread-per-object):** `mapdata`
  classes `AdministrativeRulesLoader`, `SegmentDataLoader`,
  `SegmentReferenceConverter` — they are synchronous and block the calling
  thread, so run them off the main thread; each object is single-thread only.
- `MapDownloader.fromSdkEngineAsync(...)` / `MapUpdater.fromSdkEngineAsync(...)`
  exist "to not block the UI thread" — always use the async constructors.

Source: <https://docs.here.com/here-sdk/docs/flutter-tips-supplements.md>,
<https://docs.here.com/here-sdk/docs/flutter-access-map-data-on-the-fly.md>

## 2. Initialization order (canonical)

1. `SdkContext.init(IsolateOrigin.main)` — "must be called before accessing
   SDKOptions to load necessary libraries". Does **not** init the SDK itself.
2. Build `AuthenticationMode` → `SDKOptions` (add `scope`,
   `layerConfiguration`, `catalogConfigurations`, `offlineMode`, cache paths
   here — **none of these are persisted** across re-init).
3. `await SDKNativeEngine.makeSharedInstance(sdkOptions)` — synchronous ~100 ms
   work behind `await`; once per app lifetime recommended.
4. `HereMap(onMapCreated: ...)` → in `_onMapCreated`:
   `mapScene.loadSceneForMapScheme(MapScheme.normalDay, (MapError? error) {...})`.
   Until a scene loads the map is blank.
5. Feature engines (`SearchEngine()`, `RoutingEngine()`, `LocationEngine()`,
   …) — created **after** the shared engine exists; parameterless ctor throws
   `InstantiationException`. Engine creation is "synchronous in a negligible
   amount of time" (except the async MapDownloader/MapUpdater ctors).

⚠ Init succeeds even with invalid credentials — validate with a feature call.

## 3. Widget lifecycle (Flutter-specific)

- Load scene inside `onMapCreated`; the camera may be set before or after
  scene load.
- Dispose pattern (from `flutter-integrate-here-sdk`): register an
  `AppLifecycleListener(onDetach: ...)` in `initState()` **and** clean up in
  `dispose()` of the State widget, because "Flutter may not reliably call
  dispose()" (flutter/flutter#40940).
- Do **not** dispose the SDK when a single widget dies if other widgets still
  use it; "it is also possible to init / dispose the HERE SDK for each widget
  individually" — pick one strategy and be consistent. Disposing the shared
  engine kills every engine and `HereMapController` created from it.
- For navigation: `visualNavigator.stopRendering()` **before** tearing down
  the map; set listeners to `null`; stop `LocationEngine`/`LocationSimulator`.

## 4. Engine lifecycle rules

- Engines attach to the shared `SDKNativeEngine`; recommend **one shared
  instance via `makeSharedInstance`** (it internally disposes the previous one).
- ⚠ After `await SDKNativeEngine.sharedInstance?.dispose()` every
  default-ctor engine is dead — recreate `SearchEngine()`, `RoutingEngine()`,
  etc., and recreate the `HereMap` widget after any re-init.
- Multiple engine instances need **distinct access key IDs** (cache path is
  unique per key) → same key twice throws `InstantiationException` /
  `FAILED_TO_LOCK_CACHE_FOLDER` (see `here-flutter-troubleshooting` §cache lock).
- "All engines execute their tasks asynchronously and receive their results on
  the main thread"; multiple instances of one engine may run in parallel.
- Engines do **not** need a `MapView` (headless use is supported).

Source: <https://docs.here.com/here-sdk/docs/flutter-tips-engines.md>

## 5. Callback vs listener conventions

- **Callback** = one-shot result; **listener** = recurring events.
  Multi-listener: `addXListener(...)` / `removeXListener(...)`; single
  listener: property assignment, set to `null` to stop.
- **Error/result XOR** in every engine callback: exactly one of
  `(error, result)` is non-null — `SearchError?/List<Place>?`,
  `RoutingError?/List<Route>?`, `TrafficQueryError?/List<TrafficIncident>?`,
  `MapLoaderError?/List<Region>?`, `EVSearchError?/List<EVChargingLocation>?`.
  When error is `null`, the list is guaranteed **non-null and non-empty**.
- Engine construction throws `InstantiationException`; rendering helpers throw
  `MapPolylineRepresentationInstantiationException`,
  `MapMeasureDependentRenderSizeInstantiationException`,
  `MapMarkerTextStyleInstantiationException`,
  `MapLayerBuilderInstantiationException`,
  `MapContentSettingsTrafficRefreshPeriodExceptionException` (sic, double
  "Exception"), `MapLoaderExceptionException`.
- **Wrap every callback body in defensive handling** — an app-side exception
  thrown inside an SDK callback crashes with
  `"Exception was thrown in Java and it was not handled."`
- Most async methods return a **`TaskHandle`** — use it to inspect status or
  abort (e.g. `suggestByText`).
- Suggestion callbacks have **no ordering guarantee** (a "piz" result may
  arrive before "pi" results) — never assume sequence.

## 6. Memory

- On `WidgetsBindingObserver.didHaveMemoryPressure()`:

  ```dart
  SDKNativeEngine.sharedInstance!.purgeMemoryCaches(SDKNativeEnginePurgeMemoryStrategy.full);
  // only `.full` is supported
  ```

- `SDKOptions.lowMemoryMode = true` + re-init only **after** an OS warning —
  it degrades rendering.
- "All HERE SDK classes will be garbage collected by Flutter if the instance
  is no longer referenced or set to null" — but explicitly dispose the shared
  engine on app termination; `dispose()` stops pending requests and closes
  files/DBs (it does **not** clear the on-disk cache).

Source: <https://docs.here.com/here-sdk/docs/flutter-key-concepts.md>

## 7. Callback/listener cleanup checklist

On teardown: set single listeners to `null`, call `remove*Listener` for
add/remove pairs, cancel outstanding `TaskHandle`s
(`mapDownloaderTask.cancel()`), stop engines (`locationEngine.stop()`,
`visualNavigator.route = null`), then (only at app exit)
`SDKNativeEngine.sharedInstance?.dispose(); SdkContext.release();`. Leaked
listeners keep engines (and dead widget references) alive.
