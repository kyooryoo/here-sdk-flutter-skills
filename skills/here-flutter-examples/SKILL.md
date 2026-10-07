---
name: here-flutter-examples
description: >
  Find official HERE SDK for Flutter example apps, tutorials and reference
  source code: the feature → example-app lookup table for
  heremaps/here-sdk-examples, the release-ready reference app, icons, and how
  to build and adapt an example (credentials, plugin placement). Use before
  writing feature code to copy known-good official logic, or when asked "is
  there an official example for X?".
---

# HERE SDK Flutter — Official examples lookup

Prereqs: `here-flutter-bootstrap`. Design principle: *skills teach patterns;
the official examples supply known-good code.* When an example exists, adapt
its logic instead of inventing one.

## Repos

| What | URL |
|---|---|
| Examples (all platforms/licenses) | <https://github.com/heremaps/here-sdk-examples> |
| Explore examples (Flutter) | `.../tree/master/examples/latest/explore/flutter` |
| Navigate examples (Flutter) | `.../tree/master/examples/latest/navigate/flutter` |
| Reference app (release-ready, Navigate) | <https://github.com/heremaps/here-sdk-ref-app-flutter> |
| Maneuver/lane/UI icons (open source) | <https://github.com/heremaps/here-icons> |
| Tutorials index | <https://docs.here.com/here-sdk/docs/flutter-tutorials.md> |

Layout: `examples/latest/<license>/flutter/<name>_app/` → `main.dart` (minimal
UI + credentials) + **`XYExample.dart` containing the core logic** (mostly pure
Dart, explicit callback types over lambdas).

## Building an example

1. Open the example's `main.dart` → paste your `accessKeyId`/`accessKeySecret`.
2. Unzip the SDK package → unzip the plugin TAR → **rename folder to
   `here_sdk`** → place it in the example's `plugins/` folder.
3. Attach device/emulator → `flutter run` from the example directory.
- "Unless otherwise noted, the same app can be executed with Explore or
  Navigate credentials."

## Feature → example app (★ = Navigate-only)

| Task / feature | Example |
|---|---|
| First map, init, scene load | `hello_map_app` (also has the Impeller + CarPlay/Android Auto variant `hello_map_auto_carplay_app`) |
| Camera moves, keyframe tracks | `camera_app`, `camera_keyframe_tracks_app` |
| Gestures | `gestures_app` |
| Markers/polylines/polygons/3D | `map_items_app` |
| Picking embedded POIs | `carto_poi_picking_app` |
| Map features / layers ★ | `map_features_app` |
| Custom map styles, raster layers, tile sources | `custom_map_styles_app`, `custom_raster_layers_app`, `custom_tile_source_app` |
| Traffic layers + incidents | `traffic_app` |
| Text/category search, geocode | `search_app` |
| Hybrid online/offline search ★ | `search_hybrid_app` |
| Routes, maneuvers, ETA | `routing_app` |
| Hybrid online/offline routing ★ | `routing_hybrid_app` |
| EV routing | `ev_routing_app` |
| Avoidance options ★ | `routing_with_avoidance_options_app` |
| Public transit | `public_transit_app` |
| GPS positioning, privacy notice, puck ★ | `positioning_app` |
| GPX recording / hiking ★ | `hiking_diary_app` |
| Minimal simulated turn-by-turn ★ | `navigation_quick_start_app` |
| Full navigation: ETA, TTS voice, screen-alive ★ | `navigation_app` (incl. `HEREPositioningProvider.dart`, `LanguageCodeConverter`) |
| Custom navigation UI / puck / camera ★ | `navigation_custom_app` |
| Warners (camera, school zone, …) ★ | `navigation_warners_app` |
| Spatial audio ★ | `spatial_audio_navigation_app` |
| Deviation + `returnToRoute`, maneuver panel, road shields ★ | `rerouting_app` |
| Truck guidance ★ | `truck_guidance_app` |
| Headless `Navigator` (no map), GPX trace ★ | `navigation_headless_app` |
| Offline download/update/delete ★ | `offline_maps_app` |
| Indoor/venue maps ★ | `indoor_map_app` |
| Unit testing with mocks | `unit_testing_app` |
| Camera behavior, route prefetch, e-horizon | also in ref app / see `here-flutter-navigation` |

## Ref app source pointers (production patterns)

`here-sdk-ref-app-flutter` `lib/`: `landing_screen.dart`,
`search/search_results_screen.dart`, `search/search_popup.dart`,
`search/place_details_popup.dart`, `routing/routing_screen.dart`,
`route_preferences/route_preferences_screen.dart`,
`navigation/navigation_screen.dart`, `positioning/positioning.dart`,
`download_maps/download_maps_screen.dart`.
Note: ref app supports only **major** HERE SDK versions.

## Tutorials (official order)

1. Get started with the HERE platform → <https://www.here.com/learn/blog/how-to-get-started-with-here-platform>
2. HERE Data SDK & HERE SDK on the platform → <https://www.here.com/learn/blog/here-data-sdk-here-sdk-on-the-here-platform>
3. Integrate the HERE SDK → `flutter-integrate-here-sdk`
4. Displaying maps → `flutter-maps#show-a-map`
5. Build a GPX recording app → `flutter-hiking-app-tutorial` (★ Navigate)
6. Build a navigation app → `flutter-navigation-app-tutorial` (★ Navigate)

## Verify before delivering

- [ ] Example matched to the user's license (★ examples need Navigate)
- [ ] Credentials inserted in `main.dart`, plugin placed as `plugins/here_sdk`,
      built with `flutter run` (never `pub add here_sdk`)
- [ ] Adapted code keeps the example's explicit callback types and XOR checks
- [ ] Version drift checked — example branch matches the integrated SDK
      version (`here-flutter-docs-navigation`)
- [ ] Icons sourced from `here-icons` (the SDK ships **no** maneuver icons)
