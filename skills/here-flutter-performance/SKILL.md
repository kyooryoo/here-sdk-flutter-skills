---
name: here-flutter-performance
description: >
  Audit and harden performance, data usage and binary size of a HERE SDK for
  Flutter app: engine inventory and lifecycle, map cache sizing, memory
  pressure handling, frame rate and render modes, LayerConfiguration data/OTA
  cost control, usage statistics, shrinking the release build (voice/fonts,
  ABI splits, dSYMs), credential protection, and unit testing with mocks. Use
  for review passes, "our app uses too much data/battery", or release prep.
---

# HERE SDK Flutter — Performance, cost & hardening

Prereqs: `here-flutter-bootstrap`. Cost levers marked *Navigate*;
`shared/credentials-and-licensing.md` §6 for billing/usage basics.

## 1. Engine inventory & rules (`flutter-tips-engines`)

| Engine | License | Notes |
|---|---|---|
| `SearchEngine` | both | places, suggest, geocode |
| `RoutingEngine`, `TransitRoutingEngine` | both | routes |
| `TrafficEngine` | both | incidents/flow queries |
| `OfflineSearchEngine`, `OfflineRoutingEngine` | Navigate | never go online |
| `LocationEngine`, `ConsentEngine` | Navigate | positioning; only one engine started |
| `Navigator` / `VisualNavigator` | Navigate | guidance (engine-like) |
| `DynamicRoutingEngine` | Navigate | periodic rerouting (billable) |
| `MapDownloader`, `MapUpdater`, `RoutePrefetcher` | Navigate | offline storage |
| `VenueEngine` | Navigate | indoor maps |
| `IsolineRoutingEngine` | Navigate | reach-area polygons |
| `SDKNativeEngine` | — | credentials + options (singleton) |
| `MapView`/`HereMap` | — | not an engine |

Rules: async tasks, results on main thread; multiple instances of one engine
allowed; no MapView needed for headless engines; init once via
`makeSharedInstance`; **recreate all engines after any re-init**.

## 2. Best practices checklist (`flutter-key-concepts` — the official list)

1. **Conditional SDK init** — initialize only when the map/navigation is
   actually needed (init cost + background network otherwise).
2. **Adjust frame rate** — `mapView.frameRate` (default 60): lower on low-end
   devices; **0 = deactivate automatic render cycles**; negative = no effect.
3. **Simplify visuals** — custom styles to drop layers; disable unused
   `MapFeatures` (each has a render + traffic-tile cost).
4. **Preload map data** — *Navigate*: offline regions + `RoutePrefetcher`
   instead of on-demand over the network.
5. **Limit results** — `SearchOptions.maxItems`, tight query areas.
6. **Low memory** (`WidgetsBindingObserver`):

   ```dart
   @override
   void didHaveMemoryPressure() {
     super.didHaveMemoryPressure();
     SDKNativeEngine.sharedInstance!.purgeMemoryCaches(SDKNativeEnginePurgeMemoryStrategy.full);
     // only `.full` supported; afterwards (and only after an OS warning):
     // sdkOptions.lowMemoryMode = true; await SDKNativeEngine.makeSharedInstance(sdkOptions);
   }
   ```

7. **Free resources** — null out references (GC), explicitly
   `dispose()` + recreate-on-reinit at true teardown.
8. **Remove unused voice/font files** (~11 MB max): Gradle
   `androidResources.ignoreAssetsPattern` with `!voice_package_<locale>*`
   patterns + `!DroidSansFallback.ttf:!NotoSansJP-Regular.otf` (keep **at
   least one font** or map labels disappear).
9. **Remove unused truck restriction icons** (~264 KB):
   `assets/geoviz/assets/oslo/truck_restriction/{day,night}/ui/`.
10. **ABI splits**: release APK ≈ **103 MB+** — Play limits (100 MB APK / 150 MB
    AAB) → `splits { abi { include 'x86_64','arm64-v8a' } }` or ship AAB.
11. **Render mode**: `MapRenderMode.surface` (fast) vs `.texture` (stable) vs
    hybrid composition — see `here-flutter-troubleshooting`.
12. **Protect credentials** — no plaintext; consider fetching from your server
    over SSL; anti-tampering/anti-debugging.
13. **TaskHandles** — cancel superseded searches/prefetches.
14. **Usage tracking** — `enableUsageStats(true)` before each init.
15. **iOS distribution** — do **not** strip dSYMs (symbolicated crashes
    prioritized); `PrivacyInfo.xcprivacy` ships in the xcframework.

## 3. Map cache

- `SDKOptions.cachePath` + `cacheSizeInBytes` — **≥ 256 MB (default),
  512 MB–1 GB recommended**; navigation needs zoom-14 tiles ≈ 2 km radius.
  Navigate: "a valid cache path is mandatory"; LRU when full; Explore: 1-day
  eviction. Satellite/raster cache not configurable (1-day).
- Volatile (GPU) memory: `purgeMemoryCaches(...full)`.
- Clear: `SDKCache.fromSdkEngine(engine).clearAppCache(cb)` (dispose does not).
- Cache survives power cycles; per-app, not per-device; world base tiles are
  always auto-downloaded.

## 4. Data & cost control — `LayerConfiguration` (*Navigate*)

Only listed features enabled; `rendering` is mandatory. Defaults marked ✅/❌:

| Feature | Default | Notes |
|---|---|---|
| `rendering` | ✅ | base, cannot disable |
| `detailRendering` | ✅ | buildings etc. |
| `navigation` | ✅ | omitting breaks navigation online+offline |
| `offlineSearch` | ✅ | for `OfflineSearchEngine` |
| `offlineRouting` | ✅ | for `OfflineRoutingEngine` |
| `landmarks3d` | ✅ | textured 3D landmarks |
| `ev` | ✅ | offline charging-station data |
| `offlineSearchGlobal` | ❌ | replaces `offlineSearch`; +11–16% region size; installed regions only |
| `truck` | ❌ | offline truck routes + truck map matching |
| `terrain` | ❌ | topography |
| `rdsTraffic` | ❌ | radio traffic for offline routing |
| `junctionView3x4` / `16x9` | ❌ | **up to 15 MB each** |
| `junctionSign*` | ❌ | up to 300 KB each |
| `truckServiceAttributes`, `fuelStationAttributes`, `offlineBusRouting` | ❌ | offline search/routing extras |

- `enabledFeatures` = downloaded with regions; `implicitlyPrefetchedFeatures`
  = prefetched into cache online (**subset only**; default = detailRendering +
  navigation + rendering).
- Region-sticky: set before first download; change = delete+redownload or
  `updateCatalog`.
- Other cost facts: satellite/hybrid schemes bill as **Raster Tile**
  transactions; traffic features = per-tile transactions (1–5 min refresh);
  `calculateTrafficOnRoute` + `DynamicRoutingEngine` polling are billable;
  EV billing only with `ensureReachability=true`. Usage dashboard:
  <https://platform.here.com/management/usage>; `UsageStats` per session:

  ```dart
  SDKNativeEngine.sharedInstance!.enableUsageStats(true);   // before init, every time
  List<UsageStats> stats = SDKNativeEngine.sharedInstance!.sdkUsageStats;
  // stats[i].feature.name → networkStats: sentBytes/receivedBytes/requestCounter/methodCall
  ```

## 5. Startup & rendering perf notes

- Init ~100 ms synchronous behind `await`; engines created in "negligible"
  time (except async MapDownloader/MapUpdater).
- Guidance frame rate 60 → 30 fps automatically (`guidanceFrameRate` overrides).
- Shadows: require `shadowQuality` set, cost GPU; ambient occlusion/shadows
  are **prohibited on some GPUs** (PowerVR Rogue, Adreno 730–740, Mali 32–38).
- WidgetPin (native views) lag during pan — prefer markers at scale; one
  polyline per span on huge routes is expensive (docs sample skips >5000 km).
- Cache hit behavior: world top-zoom tiles always present; partial tiles
  survive connectivity loss; stale cache has "no or only a very limited effect
  on start-up time".

## 6. Unit testing

- "all classes are fully mockable"; **SDK init not required** for unit tests:

  ```dart
  @GenerateMocks([Angle])
  void main() {
    test('angle', () {
      final mock = MockAngle();
      when(mock.degrees).thenReturn(10.0);
      expect(mock.degrees, 10.0);
    });
  }
  ```

- Example: `unit_testing_app`. Integration-test platform views via real
  example apps.

## 7. Verify before delivering — release-readiness review (agent pass)

- [ ] No plaintext credentials; scope/`LayerConfiguration`/cache options
      documented as non-persisted
- [ ] `MapFeatures` trimmed; traffic/satellite features justified; refresh
      periods set
- [ ] Frame rate / render mode chosen per target devices; Impeller decision
      recorded
- [ ] Memory pressure handler present; listeners cancelled on teardown
- [ ] ABI splits or AAB; voice/fonts/truck-icon trims applied; dSYMs kept;
      `HERE_NOTICE` + privacy text + watermark intact
- [ ] Usage stats enabled pre-init for cost monitoring
- [ ] Changelog known-issues checked for the integrated version
      (`here-flutter-docs-navigation`)
