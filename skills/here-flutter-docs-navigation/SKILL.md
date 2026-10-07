---
name: here-flutter-docs-navigation
description: >
  Verify HERE SDK for Flutter APIs, versions, and license gating against the
  live official documentation. Use whenever a class/method/option shown in
  another here-flutter-* skill is not seen before, the SDK version may have
  moved past 4.28, a changelog/known-issues check is needed, or official
  example code is required. Also use when asked what a HERE SDK feature is
  called or whether it exists.
---

# HERE SDK Flutter — Docs navigation

The guide publishes **agent-friendly plain-markdown pages** (`.md` URLs) and
`llms.txt` indexes. Prefer them over scraping HTML. When any other
`here-flutter-*` skill says "verify via `here-flutter-docs-navigation`", do it
— never invent an API.

## Indexes (start here)

| Index | URL |
|---|---|
| Site index | <https://docs.here.com/llms.txt> |
| HERE SDK project index | <https://docs.here.com/here-sdk/llms.txt> |
| **Flutter guide index (every page)** | <https://docs.here.com/here-sdk/docs/here-sdk-for-flutter-developer-guide/llms.txt> |
| Changelogs index | <https://docs.here.com/here-sdk/docs/changelogs/llms.txt> |
| Changelog (web UI) | <https://docs.here.com/here-sdk/changelog> |

## Fetching a page

Take the slug from the Flutter guide index and append `.md`:

```
https://docs.here.com/here-sdk/docs/<slug>.md
```

Examples: `flutter-routing.md`, `flutter-key-concepts.md`,
`flutter-navigation-warners.md`. The frontmatter carries `updatedAt` — use it
to judge staleness.

**Changelog tables** (the wrappers are thin; fetch the embedded files):
- Navigate: `.../docs/here-sdk-for-flutter-navigate-changelog-1.md`
- Explore: `.../docs/here-sdk-for-flutter-explore-changelog-1.md`

## What to verify, and where

| Question | Page (slug) |
|---|---|
| Does this API exist / exact signature? | Flutter guide page for the feature, then the **API Reference** (ships in the SDK package; hosted reference e.g. `here-dni.github.io/HERESDKforFlutterAPIReferenceNavigate/...`) |
| Explore or Navigate license? | `flutter-introduction-feature-list` — **Navigate-only pages carry an explicit banner**; quote it to the user |
| Is it new/beta/deprecated? | Changelog (keyword-searchable back to 4.8.2.0) + `flutter-update-version` |
| What changed when upgrading? | `flutter-update-version` + changelog for the target version |
| Known bug vs our bug? | Changelog "known issues" + `flutter-tips-troubleshooting` |
| Canonical working example | <https://github.com/heremaps/here-sdk-examples> — `examples/latest/explore/flutter/…` or `examples/latest/navigate/flutter/…`; core logic in `XYExample.dart` (`here-flutter-examples`) |
| Class list (superset of both licenses) | API Reference — license gating is per-guide-page, not per-reference |
| Current SDK version | Guide intro (`flutter-readme`) states it (e.g. **4.28.0.0**, 2026-09-29); runtime: `SDKBuildInformation.sdkVersion().versionName` |

## Rules

1. **Pin the documented version.** The current guide documents **4.28.0.0**;
   other plugin versions "may vary and you may need to adapt the source
   code." Get the integrated version via
   `SDKBuildInformation.sdkVersion().versionName`.
2. **Beta APIs have no deprecation process** — call that out when a skill
   uses a beta API (Electronic Horizon, `MapMatcher`, custom layers, style
   runtime edit, political view, prefetch, `WarnerEngine` warning model,
   `RealisticViewVectorImage`, `terrain3d`, indoor maps…). Deprecated APIs are
   kept for **two major versions / 6–9 months**.
3. **Never mix generations**: 3.x credentials don't work with 4.x; Explore
   credentials don't work with Navigate APIs (`SearchError`/
   `These credentials do not authorize access`).
4. **Docs contain occasional typos** — known ones: reverse-geocode prose says
   `searchByAddress()` (actual: `searchByCoordinates()`), Java-flavored Dart
   snippets (`LockingProcess`, `setCameraBehavior(null)`), `MapMarkerCluster.WithCounter`
   capitalization, `MapFeatures.trafficIncidents` table casing. When a snippet
   doesn't compile, check the API reference and report the doc bug via
   `here-flutter-troubleshooting` §6.
5. If the fetched page contradicts a skill in this repo, **trust the live page
   and report the drift** — then the skill should be updated.
6. Report the exact URL used when citing, so the user can spot-check.

## Support & community (for unresolved issues)

- Bug reports: <https://www.here.com/contact> (SDK version, platform, license,
  logs, repro steps) — checklist in `here-flutter-troubleshooting` §6
- Docs/example issues: <https://github.com/heremaps/here-sdk-examples/issues/new>
- Developer Slack: <https://heredev.slack.com/join/shared_invite/…>
- Release cadence: monthly — 6 minor + 4 major + 2 LTS/year; LTS supported
  6 months, hotfixes only for supported LTS
- Status: <https://status.here.com/status>
