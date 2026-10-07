---
name: here-flutter-offline
description: >
  Offline maps with the HERE SDK for Flutter: download/delete/repair regions
  with MapDownloader, update map data with MapUpdater, the global offline
  switch and pass-through features, prefetching (RoutePrefetcher/PolygonPrefetcher),
  downloading a custom area, offline search and offline routing (hybrid
  switching), and storage management. Use for any offline/caching task.
---

# HERE SDK Flutter — Offline maps

Prereqs: `here-flutter-bootstrap`. **Navigate license only** — every page
carries *"Offline maps are only available with the Navigate license."*
Async error/result XOR everywhere (`MapLoaderError?/List<Region>?`).

## 1. Storage model (two separate storages)

| Storage | Purpose | Behavior |
|---|---|---|
| **Map cache** | tiles used while panning online/headless navigation | LRU eviction when full (Navigate), 1-day (Explore); `SDKOptions.cachePath` + `cacheSizeInBytes` (≥256 MB, 512 MB–1 GB recommended) |
| **Persistent map storage** | installed offline regions | manual deletion only; `SDKOptions.persistentMapStoragePath` (default = internal app storage; never shared dirs) |

- Resolution order when rendering: cache → persistent storage → online
  download (unless offline mode). Installed region data is preferred even
  when the device is online.
- Route data is **not** cached automatically → use `RoutePrefetcher`.
- Compatibility: installed offline maps are backward **and** forward
  compatible for **one year**.

## 2. Construct the managers (async ctors — not plain constructors)

```dart
SDKNativeEngine? engine = SDKNativeEngine.sharedInstance;
if (engine == null) throw ("SDKNativeEngine not initialized.");

// "Create MapDownloader in background to not block the UI thread."
MapDownloader.fromSdkEngineAsync(engine, (mapDownloader) {
  _mapDownloader = mapDownloader;    // one MapDownloader per SDKNativeEngine
});
MapUpdater.fromSdkEngineAsync(engine, (mapUpdater) {
  _mapUpdater = mapUpdater;
});
```

Storage paths: `engine.options.cachePath`, `engine.options.persistentMapStoragePath`
(unique per access key ID).

## 3. List → download → delete regions

```dart
// 1) list (hierarchical: continents → countries → sub-regions)
_mapDownloader.getDownloadableRegionsWithLanguageCode(LanguageCode.deDe,
    (MapLoaderError? error, List<Region>? regions) {
  if (error != null) return;                 // XOR: error==null ⇒ list non-empty
  Region europe = regions!.first;
  // europe.name, europe.regionId (.id), europe.sizeOnDiskInBytes, europe.childRegions
});

// 2) download (sequential or low concurrency — parallel downloads risk OOM kills)
MapDownloaderTask task = _mapDownloader.downloadRegions(
    [region.regionId],
    DownloadRegionsStatusListener(
      (MapLoaderError? e, List<RegionId>? done) { /* complete */ },
      (RegionId id, int percentage)          { /* progress */ },
      (MapLoaderError? e)                    { /* pause; null = user pause, non-null = retried too often */ },
      ()                                     { /* resume */ }));
task.pause();  task.resume();  task.cancel();   // cancel = must restart from scratch

// 3) installed / delete
List<InstalledRegion> installed = _mapDownloader.getInstalledRegions();
// .regionId.id, .sizeOnDiskInBytes, .status.name
_mapDownloader.deleteRegions(ids, (MapLoaderError? e, List<RegionId>? deleted) { ... });
```

- Pause is auto-retried by the SDK on bad network; too often → error in
  `onPause`, only a user `resume()` continues. Calling `downloadRegions()`
  again for a paused region = same as `resume()`.
- Delete while a download is pending → `INTERNAL_ERROR`; while running →
  `NOT_READY`. Cannot delete while any download is paused.
- **Custom area** (not a predefined region):
  `_mapDownloader.downloadArea(GeoPolygon.withGeoBox(geoBox), DownloadRegionsStatusListener(...))`
  → one unique `RegionId` (store a human-readable name for later deletion);
  storage is deduplicated across overlapping downloads.
- Best practice: never download/update in the background — prompt the user to
  `pause()`/`cancel()`, offer `resume()`, keep screen awake via platform code.

## 4. Repair after crash

```dart
if (_mapDownloader.getInitialPersistentMapStatus() != PersistentMapStatus.ok) {
  _mapDownloader.repairPersistentMap((MapLoaderRepairError? e) { ... });
}
```

- `pendingUpdate` status can't be repaired → needs an update instead.
- Last resort: `deleteRegions()` + `SDKCache.fromSdkEngine(engine).clearAppCache(cb)`
  + restart. (Disposing the engine does **not** clear the cache.)

## 5. Updates (`MapUpdater`)

```dart
MapVersionHandle v = _mapUpdater.getCurrentMapVersion();
// v.stringRepresentation(",") → "47.47,47.47"  (cache.offline[,japanCache.japanOffline])

_mapUpdater.retrieveCatalogsUpdateInfo((MapLoaderError? e, List<CatalogUpdateInfo>? infos) {
  // .installedCatalog.catalogIdentifier.hrn/.version, .latestVersion  ← size known BEFORE updating
  _mapUpdater.updateCatalog(infos!.first, CatalogUpdateProgressListener(
      (RegionId id, int pct) {}, (MapLoaderError? e) {}, (MapLoaderError? e) {}, () {}));
});
```

- Weekly OCM map updates; each SDK version pins a map version; **incremental
  updates** by default (~8 versions of benefit, then a bigger download).
- ⚠ **After `updateCatalog()` you must call `getDownloadableRegions()`** to
  refresh internal catalog data (before any new download/update/delete).
- Cannot force a version by delete+re-download; **cannot downgrade installed
  regions** (`CatalogVersionHint.specific(n)` only affects cache when nothing
  is installed).
- ⚠ `MapDownloader`/`MapUpdater` features may be temporarily unavailable
  during an update — retry after completion.

## 6. Global offline switch & pass-through

```dart
// runtime:
SDKNativeEngine.sharedInstance?.isOfflineMode = true;   // blocks ALL online SDK requests
// or at init (sufficient if never toggled):
sdkOptions.offlineMode = true;

// allow selected data through while "offline":
engine.passThroughFeatures = <PassThroughFeature>{
  PassThroughFeature.trafficTilesFlow, PassThroughFeature.trafficTilesIncidents};
```

⚠ Even with regions installed and the device online, the online engines never
fall back to offline data: **the online `SearchEngine`/`RoutingEngine` "will
not return results when executed offline"** and offline engines never go
online — swap engines explicitly:

```dart
final engine = useOnline ? _onlineRoutingEngine : _offlineRoutingEngine;
// both implement RoutingInterface — clean hybrid switch (see routing_hybrid_app)
```

## 7. Offline search & routing

- `OfflineSearchEngine()` — same methods as `SearchEngine`; results may differ;
  offline place IDs ≠ online IDs; free-form text reach ≈ **62.5 km radius**;
  don't rely on cache alone. Optional **beta** index:
  `OfflineSearchEngine.setIndexOptions(engine, OfflineSearchIndexOptions()..enabled = true, listener)`
  (~5% storage, slows all storage-mutating ops).
- `OfflineRoutingEngine()` — no waypoint limit; **routes have empty maneuver
  lists** (take them from the navigator); EV and transit routes **not
  supported** (`RoutingError.invalidParameter`); truck routes need
  `LayerConfigurationFeature.truck`.
- Missing map data anywhere on the route ⇒ route calculation fails.

## 8. Prefetch (beta)

> "this is a beta release of this feature… Related APIs may change for new
> releases without a deprecation process."

```dart
RoutePrefetcher routePrefetcher = RoutePrefetcher(SDKNativeEngine.sharedInstance!);
routePrefetcher.prefetchAroundRouteOnIntervals(_visualNavigator!);  // call once after guidance starts
// corridor defaults: 10 km long × 5 km wide; first prefetch after 9 km; idempotent
PolygonPrefetcher polygonPrefetcher = PolygonPrefetcher(engine);
polygonPrefetcher.prefetch(GeoPolygon.withGeoCircle(GeoCircle(coords, radius)),
    PrefetchStatusListener((int pct) {}, (MapLoaderError? e) {}));
```

- Prefetched data is cache (evictable), not permanent; "increases network
  traffic continuously during guidance"; insufficient cache space → error.

## 9. Cost control — `LayerConfiguration` (Navigate)

Set **before init**; only listed features are enabled (everything else is
disabled; `rendering` cannot be disabled):

```dart
LayerConfiguration lc = LayerConfiguration([LayerConfigurationFeature.detailRendering,
  LayerConfigurationFeature.rendering, LayerConfigurationFeature.navigation,
  LayerConfigurationFeature.offlineSearch, LayerConfigurationFeature.offlineRouting]);
sdkOptions.layerConfiguration = lc;
```

- ⚠ The config at **first download** sticks to that region — changing it needs
  delete+re-download or `updateCatalog`. `implicitlyPrefetchedFeatures` must be
  a **subset** of `enabledFeatures`. Defaults differ between the two lists.
- Big levers: `junctionView*` (up to 15 MB each), `truck` (default off),
  `terrain`, `offlineSearchGlobal` (replaces `offlineSearch`; +11–16% region
  size). Full table: `here-flutter-performance`.
- Changing it at runtime = dispose → re-init → rebuild `HereMap` with a
  `UniqueKey()` → recreate `MapDownloader`/`MapUpdater`/engines.

## Verify before delivering

- [ ] Navigate license confirmed; async `fromSdkEngineAsync` constructors used
- [ ] XOR checked on every `MapLoaderError` callback; progress/pause/resume/
      cancel wired to real UI
- [ ] Downloads sequential/throttled; pause/cancel offered on backgrounding;
      storage sizes surfaced to the user
- [ ] `getDownloadableRegions()` called after every `updateCatalog()`;
      `repairPersistentMap` on non-OK status
- [ ] Online/offline engines swapped explicitly (never assumed fallback);
      offline routes' empty maneuvers handled
- [ ] `LayerConfiguration` set before init and documented (region-sticky);
      prefetch beta flagged
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
