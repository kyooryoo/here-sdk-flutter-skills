# HERE SDK for Flutter — AI Agent Skills

A **skill set that teaches AI coding agents to build mobile apps with the HERE
SDK for Flutter (4.x)** — from project bootstrap to turn-by-turn navigation,
offline maps, cost/performance hardening, and official-doc verification.

The skills encode the ~75 pages of the official HERE SDK for Flutter
Developer Guide (v4.28.0.0), collected by walking the guide's machine-readable
indexes (`llms.txt`), plus the non-obvious contracts that generated code
usually gets wrong: the `SdkContext` → `SDKNativeEngine` init sequence, local
TAR plugin delivery (no pub.dev), error/result XOR callbacks, license gating
(Explore vs Navigate), the `MapFeatures` scene-load reset, the Android
privacy-notice gate before `LocationEngine.start()`, and the
`WarnerEngine.setEnabledWarnings` gate.

---

## 1. Description

| | |
|---|---|
| **Purpose** | Enable an AI agent to help developers create HERE SDK Flutter apps correctly, following official HERE guides and best practices, without re-deriving the SDK's quirks on every task. |
| **Target SDK** | HERE SDK for Flutter **4.28.0.0** (Explore & Navigate licenses; Android + iOS — no web/desktop) |
| **Format** | Agent skills: one `SKILL.md` per topic (router frontmatter + instructions + verify checklist) with a shared correctness core in `shared/` |
| **Coverage** | Project setup & credentials · maps/camera/gestures/items/styles/indoor · search & geocoding · routing (car/truck/EV/isolines) · traffic · positioning · turn-by-turn navigation (voice, warners, lanes, deviation) · offline maps · performance & data-cost hardening · troubleshooting · official examples lookup · live-docs verification |
| **Design principle** | *Skills teach patterns; the official examples supply known-good code; `here-flutter-docs-navigation` verifies signatures.* Every skill ends with a **verify checklist** derived from documented failure modes. |

## 2. How it was created

### Phase 1 — Research

1. Started from `https://docs.here.com/here-sdk/docs/flutter-readme` and
   followed the docs' agent indexes (`llms.txt` at site → project → guide
   level) to map all ~75 Flutter guide pages.
2. Fanned out **6 parallel research agents**, each fetching every `.md` page
   in a documentation cluster:
   - setup/licensing/integration (requirements, editions, credentials, TAR
     plugin, Impeller, examples, tutorials, coverage)
   - maps & customization (schemes, camera, gestures, items, picking, styles,
     custom layers, customization, indoor/venues, CarPlay/Android Auto)
   - search / routing / traffic (13 pages incl. EV, isolines, TrafficEngine)
   - positioning & navigation (16 pages incl. permissions, warners, lanes,
     e-horizon, truck, deviation, voice, tutorials)
   - offline / engines / best practices / optimization / versioning
   - troubleshooting / FAQ / billing / legal / changelogs
3. Notes were cross-checked for **naming traps** (no `HereSdk.init()` — it's
   `SdkContext.init` + `makeSharedInstance`; widget is `HereMap` with
   `HereMapController`; offline API is `MapDownloader`/`MapUpdater`, not
   `PackageManager`; `returnToRouteWithTraveledDistance`, not `returnToRoute()`;
   reverse geocoding is `searchByCoordinates()`, not `searchByAddress()`).

### Phase 2 — Build

Skills were written to the conventions of the sibling skill sets in this
workspace (`here-maps-sdk-android`, `here-maps-web`, `mapbox-agent-skills`):
router-ready YAML frontmatter, task-shaped boundaries, shared correctness
core, and a verify-before-delivering checklist on every skill. Facts that
could not be confirmed on a fetched page are marked "verify via
`here-flutter-docs-navigation`" instead of being guessed; documented typos and
inconsistencies are flagged rather than copied.

### Phase 3 — Validation

- Frontmatter (`name`, `description`) present on all 12 skills.
- Every API name used in a skill appears in the fetched research notes from
  the cited official page; no invented APIs.
- Cross-references between skills and `shared/*` resolved.

## 3. What's inside

```
here-maps-sdk-flutter/
├── README.md                          ← this file
├── shared/                            ← correctness core, referenced by all skills
│   ├── credentials-and-licensing.md   delivery (TAR, no pub.dev), OAuth keys, Explore vs Navigate, scope, legal
│   └── threading-and-lifecycle.md     main-thread law, init order, widget/engine lifecycle, callbacks, memory
└── skills/
    ├── here-flutter-bootstrap/        [Tier 0] TAR plugin, init, HereMap, scene, Impeller, render modes, scope
    ├── here-flutter-maps/             [Tier 1] schemes, camera, gestures, items, picking, features, styles, indoor
    ├── here-flutter-search/           [Tier 1] text/suggest/geocode/category, Place results, offline/W3W/EV search
    ├── here-flutter-routing/          [Tier 1] routes, modes, options, avoid, truck/EV, isolines, refresh, advanced
    ├── here-flutter-traffic/          [Tier 1] flow/incident layers, TrafficEngine, traffic-on-route, costs
    ├── here-flutter-positioning/      [Tier 1] LocationEngine, permissions, privacy gate, accuracy, puck, background
    ├── here-flutter-navigation/       [Tier 1] VisualNavigator, voice/TTS, deviation, warners, lanes, truck, e-horizon
    ├── here-flutter-offline/          [Tier 1] regions, updates, repair, prefetch, offline search/routing, offline mode
    ├── here-flutter-performance/      [Tier 2] engines, cache, memory, frame rate, LayerConfiguration, APK size, testing
    ├── here-flutter-troubleshooting/  [Tier 2] symptom→fix table, cache lock, logging, symbolication, bug reports
    ├── here-flutter-examples/         [Tier 3] feature → official example-app lookup table + tutorials
    └── here-flutter-docs-navigation/  [Tier 3] llms.txt/.md page fetching, changelog, license/version checks
```

**Totals:** 12 skills · 2 shared docs.

### Skill inventory

| Skill | Use it when the user wants to… |
|---|---|
| `here-flutter-bootstrap` | create/init a HERE Flutter project, add a map, fix a blank map at startup (always first) |
| `here-flutter-maps` | switch theme/scheme, animate camera, handle gestures/taps, draw markers/shapes, style the map, indoor maps |
| `here-flutter-search` | search addresses/POIs, autocomplete, geocode, search along a route |
| `here-flutter-routing` | calculate/display routes, truck/EV routing, waypoints, avoid options, ETA, isolines |
| `here-flutter-traffic` | show traffic, query incidents/flow, refresh route ETA |
| `here-flutter-positioning` | get GPS locations, show the blue dot, background location, location simulation |
| `here-flutter-navigation` | turn-by-turn guidance, voice, rerouting, warners, lane guidance, truck nav |
| `here-flutter-offline` | download maps, work offline, manage storage, hybrid online/offline engines |
| `here-flutter-performance` | audit performance/data costs/binary size, harden for release, unit-test with mocks |
| `here-flutter-troubleshooting` | debug an error, collect logs, symbolicate, file a HERE bug report |
| `here-flutter-examples` | find the official example app or tutorial for a feature |
| `here-flutter-docs-navigation` | verify an API, check license gating, check changelog/version |

## 4. How to use

### Prerequisites (for generated apps)

- HERE account + OAuth credentials → `shared/credentials-and-licensing.md`
- SDK package downloaded from the platform portal and the plugin folder
  renamed to `here_sdk` (path dependency — **not on pub.dev**).
- License choice: **Explore** (online maps/search/routing) vs **Navigate**
  (adds navigation, positioning, offline, warners, indoor, e-horizon). Every
  skill that uses Navigate-only APIs says so.
- Flutter ≥ 3.44.8 / Dart ≥ 3.12.2 · Android minSdk 24 / compileSdk 36 ·
  iOS 15.2 / Xcode ≥ 16.2.

### Loading the skills in an agent

- **Claude Code / similar**: copy `skills/` into the project's skills
  directory (e.g. `.claude/skills/`), plus `shared/`.
- **OpenCode**: add the paths as skill sources in config; skills load on
  description match.
- **Custom agents**: parse frontmatter — `description` is the router key;
  load the full `SKILL.md` on match; resolve `shared/*` references on demand.

Recommended loading order:

1. `here-flutter-bootstrap` — always (defines the shared init).
2. Domain skill(s) matching the task (maps · search · routing · traffic ·
   positioning · navigation · offline).
3. `here-flutter-examples` before writing feature code — adapt official
   example logic when one exists.
4. `here-flutter-docs-navigation` whenever a skill says "verify", the SDK
   version differs from 4.28, or license gating matters.
5. `here-flutter-performance` + `here-flutter-troubleshooting` for review
   passes and runtime issues.

### Example prompts the skills are tuned for

- "Create a Flutter app showing a HERE map with day/night switching."
- "Search nearby restaurants and drop markers for the results."
- "Draw a truck route with axle weights and show the maneuvers list."
- "Add turn-by-turn navigation with voice prompts and speed warnings."
- "Let users download Europe for offline use, then navigate offline."
- "Our app uses too much data — audit it." / "Blank map, how do I debug?"

## 5. Suggestions & future work

- **Golden-prompt evaluation set** (~20 tasks × expected checklist outcomes)
  run against an agent with/without the skills; fold misses into checklist
  lines.
- **Automated staleness check**: a small job comparing each fetched page's
  `updatedAt` frontmatter against the build date to flag skills needing
  review (the `.md` pages expose it — ideal for CI).
- **Harvested recipe catalog**: like `here-maps-web`'s `here-task-recipes`,
  a script could index the `here-sdk-examples` repo (tree API → per-example
  API-symbol index) for code-first lookups; the current `here-flutter-examples`
  table is hand-curated.
- **Version-scoped skills** if HERE ships a breaking release; pin via a
  version banner in frontmatter and let `here-flutter-docs-navigation` detect
  drift. Note 4.28 deprecations to fold in: `MapMarker.iconOverlapAllowed`,
  several indoor routing error members (now route notices),
  `EVConsumptionModel`/`EVCarOptions` (planned removal).
- **Split navigation skill** (`voice`/`warners`/`lanes`) if usage data shows
  the page grows unwieldy; it's the largest skill today.
- **Widget-layer recipes**: deeper Flutter patterns (state hoisting for
  `HereMapController`, `ViewModel`-scoped engines, `permission_handler`
  wiring, `flutter_tts` integration) are only referenced — promote if demand
  shows.
- Regional/vertical deep-dives (Japan data/OCM catalogs, truck amenities,
  indoor routing) are cross-referenced only — promote if demand shows.

---

*Built from the official HERE SDK documentation (subject to the
[HERE Documentation License](https://legal.here.com/en-gb/terms/documentation-license));
example links point to the official `heremaps` GitHub repositories.*
