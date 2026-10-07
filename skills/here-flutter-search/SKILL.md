---
name: here-flutter-search
description: >
  Search places and addresses with the HERE SDK for Flutter: text search,
  forward/reverse geocoding, category search, autocomplete/suggest, search
  along a route, Place/Suggestion handling, offline search, what3words, and EV
  charging enrichment. Use for any POI/address/geocode/autocomplete task in a
  HERE Flutter app.
---

# HERE SDK Flutter — Search & geocoding

Prereqs: `here-flutter-bootstrap`. Explore license covers all online search;
`OfflineSearchEngine`, `W3WSearchEngine`, my-places are **Navigate only**
(`shared/credentials-and-licensing.md` §4). All callbacks arrive on the main
thread with **error/result XOR** semantics.

## Engines

```dart
try {
  _searchEngine = SearchEngine();                 // online
  // _offlineSearchEngine = OfflineSearchEngine(); // Navigate only
  // _w3wSearchEngine = W3WSearchEngine();         // Navigate only
} on InstantiationException {
  throw Exception("Initialization of SearchEngine failed.");
}
```

- Each search is async; online connection required (offline engine never
  goes online). Not initializable in `Application.onCreate()`.
- EV enrichment: `_searchEngine.setEVInterface(evSearchEngine)` —
  `EVSearchEngine` needs the **EV Charge Points API v3 entitlement on top of
  Navigate** (else `SearchError.forbidden`).

## The four entry points

```dart
SearchOptions searchOptions = SearchOptions();
searchOptions.languageCode = LanguageCode.enUs;
searchOptions.maxItems = 30;

// 1) Text
TextQueryArea queryArea = TextQueryArea.withBox(viewportGeoBox);   // or .withCenter(coords)
TextQuery query = TextQuery.withArea(queryString, queryArea);
_searchEngine.searchByText(query, searchOptions,
    (SearchError? error, List<Place>? list) async { ... });

// 2) Address / forward geocode (area center ranks nearby results higher)
AddressQuery addressQuery = AddressQuery.withAreaCenter(queryString, geoCoordinates);
_searchEngine.searchByAddress(addressQuery, searchOptions, (error, list) { ... });

// 3) Coordinates / reverse geocode (worldwide, typically 1 result)
_searchEngine.searchByCoordinates(geoCoordinates, searchOptions, (error, list) { ... });

// 4) Category
List<PlaceCategory> cats = [PlaceCategory(PlaceCategory.eatAndDrink),
                            PlaceCategory(PlaceCategory.shoppingElectronics)];
CategoryQueryArea area = CategoryQueryArea.withCenter(GeoCoordinates(52.52, 13.41));
CategoryQuery catQuery = CategoryQuery.withCategoriesInArea(cats, area);
_searchEngine.searchByCategory(catQuery, searchOptions, (error, list) { ... });

// Autocomplete
_searchEngine.suggestByText(TextQuery.withArea("piz", TextQueryArea.withCenter(center)),
    searchOptions, (SearchError? error, List<Suggestion>? list) { ... });
```

- XOR law: `SearchError` and `List<Place>` "can never be null at the same time
  - or non-null at the same time"; **error == null ⇒ list non-null and
  non-empty**.
- Empty query string is an **error** — fail fast in app code.
- `suggestByText` returns a `TaskHandle` (cancel pending calls); callback
  order is **not guaranteed** across keystrokes — debounce and drop stale
  responses by query.
- The SDK ships **no autocomplete UI** — "The HERE SDK does not provide any UI
  or a fully integrated auto completion solution."
- `SearchError` is per-engine: watch `authenticationFailed`, `forbidden`
  (missing license), `requestLimitReached`, `layersNotDownloaded`.

## Query-area options

| Type | Constructors / notes |
|---|---|
| `TextQueryArea` | `withBox(GeoBox)`, `withCenter(GeoCoordinates)`; corridor/circle variants exist (verify via `here-flutter-docs-navigation`) |
| `CategoryQueryArea` | `withCenter(GeoCoordinates)`, `withCorridorAndCenter(GeoCorridor, GeoCoordinates)` |
| `GeoCorridor` | `GeoCorridor(route.geometry.vertices, halfWidthInMeters)` — round caps; bigger = cheaper but less precise |
| Search along route | category query + corridor (see below); **beta** |

```dart
// Search along a route
GeoCorridor routeCorridor = GeoCorridor(route.geometry.vertices, 200);
CategoryQueryArea qArea = CategoryQueryArea.withCorridorAndCenter(
    routeCorridor, hereMapController.camera.state.targetCoordinates);
CategoryQuery q = CategoryQuery.withCategoryInArea(
    PlaceCategory(PlaceCategory.businessAndServicesEvChargingStation), qArea);
_searchEngine.searchByCategory(q, searchOptions, (e, items) { ... });
```

- Results outside the requested area rank lower "except for relevant global
  results". `PlaceCategory` statics (700+): `PlaceCategory.eatAndDrink`,
  `.shoppingElectronics`, `.businessAndServicesEvChargingStation`; custom IDs
  are 3-level numeric (`xxx-xxxx-xxxx`).
- Place chains: `categoryQuery.includeChains = [PlaceChain("1566")]`.

## Place & Suggestion

- `Place`: `.geoCoordinates` (⚠ nullable for suggestions), `.title`,
  `.address` → `Address` (`.addressText` = readable one-liner), `.id` (HERE
  Place ID), `.accessPoints` (vehicle-reachable entry points), `.details`
  (`.images`, `.evChargingPool`, `.evChargingLocation`, truck amenities).
- `Suggestion`: `.title` always; `.place` **nullable** — generic terms like
  "disco" have no place.
- Attach to a marker: custom `CustomMetadataValue` implementation stored via
  `marker.metadata?.setCustomValue("key", SearchResultMetadata(place))`,
  retrieved with `getCustomValue("key")`.
- Reverse geocode pattern: `maxItems = 1` + read `list!.first.address.addressText`.
- ⚠ Doc trap: the geocoding page's prose says `searchByAddress()` for reverse
  geocoding — the actual API is **`searchByCoordinates()`**.

## Offline search (Navigate only)

- Same method names on `OfflineSearchEngine`; results may differ (older data,
  local). **The online engine returns nothing offline even when data is
  cached** — swap engines explicitly:

  ```dart
  final engine = useOnline ? _searchEngine : _offlineSearchEngine;
  engine.searchByText(query, searchOptions, cb);
  ```

- Reach: free-form text finds places "at least within a radius of 62.5 km";
  offline place IDs differ from online HERE Place IDs.
- My places (bring-your-own offline POIs):
  `GeoPlace.makeMyPlace(name, coords)` → `myPlaces.addPlaces(...)` →
  `offlineSearchEngine.attach(myPlaces, cb)`.
- Hybrid example: `search_hybrid_app`.

## what3words (Navigate only)

```dart
_w3wSearchEngine.searchByWords("index.home.boat", (error, W3WSquare square) { ... });
_w3wSearchEngine.searchByCoordinates(coords, "en", (error, square) { ... });
// square.words, square.square.southWestCorner / .northEastCorner
```

## EV enrichment

```dart
EVSearchOptions evOptions = EVSearchOptions();
evOptions.additionalFeatures = [EVChargingLocationFeature.energyMix, EVChargingLocationFeature.openingHours];
_evSearchEngine.setOptions(evOptions);
_searchEngine.setEVInterface(_evSearchEngine);
_searchEngine.setCustomOption("browse.show", "ev");   // pools w/ availability
// → place.details.evChargingPool.chargingStations[] (availableConnectorCount, lastUpdated, ...)
```

- Missing license → `SearchError.forbidden`; TripAdvisor web images:
  `setCustomOption("discover.show", "tripadvisor")` (empty list without
  credentials).
- Truck amenities: closed-alpha `setCustomOption("show", "truck")` + extra
  license (online); `OfflineSearchEngine` needs no extra license.

## Verify before delivering

- [ ] Error branch checked **before** touching the result list (XOR); list
      assumed non-empty when error is null
- [ ] `SearchOptions.languageCode`/`maxItems` set deliberately; empty-query
      guarded
- [ ] Suggest debounce + stale-response drop (out-of-order callbacks); cancel
      via `TaskHandle` where sensible
- [ ] Area/corridor types match the intent (viewport box vs center vs
      route corridor)
- [ ] Navigate-only features flagged: offline search, W3W, my places, truck
      amenities, EV entitlement
- [ ] No UI invented — autocomplete/lists are app-built
- [ ] Signatures beyond this page verified via `here-flutter-docs-navigation`
