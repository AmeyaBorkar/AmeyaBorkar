# proximap

> A key-free geospatial toolkit for **places, proximity, and amenities** built on OpenStreetMap data. It is **not** a map-rendering plugin (no Leaflet/MapLibre/Obsidian/Figma host). It is a TypeScript engine that geocodes a place, fetches points of interest from OSM, and answers higher-order questions: rank nearby amenities by distance or real travel time, score walkability, detect amenity gaps, list what's reachable in N minutes, plan a shortest multi-stop errand, and compare neighbourhoods. Shipped as three published npm packages: a library (`@proximap/core`), a CLI (`@proximap/cli`), and an MCP server (`@proximap/mcp`) for AI agents. Current release: **v1.0.1**.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/proximap |
| **Visibility** | Public |
| **Category** | web |
| **Primary language(s)** | TypeScript (100% of source; zero runtime deps in core) |
| **Local path** | `C:\Users\ameya\Documents\MapsPlugin` |
| **Default branch** | `main` (local `HEAD` = `origin/main`) |
| **Lines of code (computed)** | 5,171 source TS + 1,956 test TS = **7,127 LOC of TypeScript**, plus 39 lines across 4 build/test configs. About 1,411 lines of Markdown docs across 23 files. Counted over `git ls-files`, so node_modules, dist and `package-lock.json` are excluded. |
| **Source files (computed)** | 31 non-test `.ts` modules (excludes the 4 configs: 3 `tsup.config.ts` + `vitest.config.ts`) + 22 `.test.ts` files = 53 TS files; **139 test cases**. 102 tracked files in total. |
| **Package / host platform** | npm-workspaces monorepo, Node ≥ 20. Three published packages, all at **1.0.1**: `@proximap/core` (library), `@proximap/cli` (bin `proximap`), `@proximap/mcp` (bin `proximap-mcp`, stdio MCP server). Root pkg `proximap-monorepo` (1.0.1) is private. |
| **Key dependencies** | core: **none at runtime** (platform `fetch`). cli: `commander` ^15, `picocolors` ^1. mcp: `@modelcontextprotocol/sdk` ^1.29, `zod` ^4. dev: `tsup`, `vitest`, `typescript` ^6, `prettier`. |
| **License** | MIT © Ameya Borkar |
| **Last commit** | `0d4ba89` "chore(release): v1.0.1" (2026-06-18). First commit `5146e59` on 2026-06-16; **36 commits total** (29 on 2026-06-16, 7 on 2026-06-17/18). Tags `v1.0.0`, `v1.0.1`. No commits since 2026-06-18. |

**Since the first version of this doc (2026-06-16, v1.0.0):** seven commits, one patch release (v1.0.1), six correctness fixes, seven new regression tests (132 → 139), and +109 LOC of TypeScript (7,057 → 7,166 counting configs, the June basis). Production source grew by 20 lines and tests by 89. No new commands, MCP tools, or library functions were added. *Recount note:* the June doc reported "33 non-test modules (excludes the 3 configs)." Recounting the same commit (`2a849d4`) gives 31 modules + 4 configs (5,151 + 39 = 5,190 lines). The June LOC total already included the configs, and the module count was off by two.

---

## What it actually is (be concrete)

The local working directory is named "MapsPlugin," but **proximap is not a plugin for any map host.** There is no `manifest.json` and no Obsidian/Figma manifest. There is no Leaflet/MapLibre/Mapbox GL dependency, and nothing renders tiles. It is a **headless geospatial reasoning engine** with three delivery surfaces.

The core abstraction is three small provider interfaces (`types.ts`):

- **`GeocodingProvider`**: `geocode(query)` plus an optional `reverse(latlng)`. Default impl: `NominatimGeocoder` (OSM Nominatim).
- **`PlacesProvider`**: `findNearby(center, { radiusMeters, selectors })`. Default impl: `OverpassPlacesProvider` (OSM Overpass API). `DatasetPlacesProvider` serves offline snapshots.
- **`RoutingProvider`**: `matrix(origin, targets, mode)` plus an optional `isochrone(...)`. Default impl: `HaversineRoutingProvider` (straight-line, no network). The real engines are `ValhallaRoutingProvider` and `OsrmRoutingProvider`.

The headline entry point is `findNearbyAmenities(query, options)` in `packages/core/src/nearby.ts`. It resolves an origin (a place name, a `"lat,lng"` string, or a `LatLng`) and fetches POIs. It classifies each POI into one of **16 normalized categories** and applies facet and open-now filters. It then ranks by distance (haversine) or by travel time (a routing matrix with a haversine fallback). Everything else **composes** that same pipeline and the same providers: `walkabilityScore`, `detectGaps`, `reachableAmenities`, `compareLocations`, `planErrands`, and `snapshotArea`.

The code and docs show three design choices:

- **Key-free defaults.** OSM Nominatim, Overpass, and the FOSSGIS Valhalla instance need no keys or billing.
- **Honesty about data.** A missing amenity is reported as "not found in OSM." Scores carry a `confidence`. An ambiguous place name returns candidates instead of a guess.
- **Agent-native output.** Ordering is deterministic (ties are broken by a stable `id`). There is an `explain` mode and a `concise` MCP payload.

## Architecture & how it's structured

npm-workspaces monorepo (`workspaces: ["packages/*"]`), Node ≥ 20, ESM-first. Annotated tree (node_modules/dist omitted):

```
MapsPlugin/
├─ package.json              root: "proximap-monorepo" (private, 1.0.1), build/test/check scripts
├─ tsconfig.base.json        strict TS6; ignoreDeprecations "6.0" (tsup d.ts injects baseUrl)
├─ vitest.config.ts          include packages/*/src/**/*.test.ts, node env
├─ ARCHITECTURE.md · ROADMAP.md · CHANGELOG.md · CONTRIBUTING.md · README.md
├─ docs/proposals/           01–12 per-feature design specs + README (the researched backlog)
├─ examples/demo.ps1         PowerShell "highlight reel" running the built CLI against live OSM
├─ packages/
│  ├─ core/                  @proximap/core — the engine (entry: src/index.ts)
│  │  └─ src/
│  │     ├─ types.ts         domain model: LatLng, Place, Poi, RankedPoi, CATEGORIES, provider interfaces
│  │     ├─ geo.ts           haversine, formatDistance/Duration, parseCoordinates
│  │     ├─ categories.ts    categorize(tags) → one of 16 categories; CATEGORY_LABELS
│  │     ├─ taxonomy.ts      NL term resolver ("coffee"→tag union), Overpass filter builder, typo suggest
│  │     ├─ quality.ts       completenessOf, lastVerifiedOf, dedupePois
│  │     ├─ hours.ts         hand-written opening_hours evaluator (isOpenAt)
│  │     ├─ http.ts          fetch wrapper: RateLimiter, retry/backoff, InMemoryCache, HttpError
│  │     ├─ filters.ts       FacetFilters → predicates; accessibleScorer
│  │     ├─ ranking.ts       rankByProximity (default scorer, stable tie-break)
│  │     ├─ routing.ts       RoutingProvider iface, HaversineRoutingProvider, circle/point-in-polygon
│  │     ├─ origin.ts        resolveOrigin (geocode | parse "lat,lng" | reverse)
│  │     ├─ proximity.ts     nearestMatchingPoi (shared by gaps + walkability)
│  │     ├─ disambiguate.ts  disambiguateLocation (ambiguity guard)
│  │     ├─ nearby.ts        findNearbyAmenities — HEADLINE entry point
│  │     ├─ reachable.ts     reachableAmenities (isochrone or matrix threshold)
│  │     ├─ gaps.ts          detectGaps (what's missing)
│  │     ├─ walkability.ts   walkabilityScore (0–100 + breakdown + confidence)
│  │     ├─ compare.ts       compareLocations (N-location scorecard)
│  │     ├─ errands.ts       planErrands (Generalized-TSP, Held-Karp DP)
│  │     ├─ export.ts        toGeoJSON / toCSV + ODbL attribution
│  │     ├─ snapshot.ts      snapshotArea + DatasetPlacesProvider (offline)
│  │     └─ providers/       nominatim · overpass · valhalla · osrm (+ index)
│  ├─ cli/                   @proximap/cli — bin "proximap" (commander)
│  │  └─ src/ index.ts (command wiring) · render.ts (picocolors terminal output)
│  └─ mcp/                   @proximap/mcp — bin "proximap-mcp" (MCP stdio server)
│     └─ src/ index.ts (8 registerTool calls) · payload.ts (agent-flattened payloads)
├─ apps/web/README.md        Web UI — PLANNED, not implemented (placeholder dir)
└─ python/README.md          PyPI port — PLANNED, not implemented (empty src/proximap/ dir, no .py files)
```

**Entry points:**
- Library: `packages/core/src/index.ts`, which re-exports everything and sets `VERSION = '1.0.1'`.
- CLI: `packages/cli/src/index.ts`, which calls commander's `program.parseAsync()`.
- MCP: `packages/mcp/src/index.ts`, an `McpServer` over `StdioServerTransport`.

The CLI and MCP packages each keep their own hard-coded `VERSION = '1.0.1'`, bumped by hand in the release commit.

## Code walkthrough (the important parts)

**`nearby.ts` — `findNearbyAmenities()`.** The pipeline runs in this order:
1. Resolve natural-language category terms to selectors. `resolveSelectors` throws with typo suggestions on unknown terms.
2. Resolve the origin with `resolveOrigin`.
3. Fetch POIs with `places.findNearby`.
4. Keep only POIs that match the selectors, then apply the compiled facet predicates.
5. When `open` is set, drop POIs that are confirmed closed. Open and unknown-hours POIs are kept, so unknown hours never cause a silent drop.
6. Rank.

Travel-time ranking (`rankByTravelTime`) first trims to the nearest `TRAVEL_MATRIX_CAP = 80` candidates by haversine, which limits matrix size and load on the public engine. It then calls `routing.matrix`. **If the engine errors, it falls back to haversine** and sets `routing.fellBack`. `explain` attaches a deterministic `rankingReason` such as "closest open cafe, 240 m", with proper English ordinals.

**`categories.ts` / `taxonomy.ts` — the two-way vocabulary.** `categorize(tags)` handles the reading direction. It maps OSM tags to one of 16 categories in priority order: `amenity` → `shop` → `tourism` → `healthcare` → `leisure` → `railway`/`public_transport`/`highway`/`aeroway` → `office`. The raw value is kept as `kind`.

`taxonomy.ts` handles the query direction. Its `TERMS` table maps everyday words ("chemist", "petrol", "cashpoint") and their synonyms to OSM `CategorySelector[]`. The selectors are deduplicated and rolled up to a category. `suggestCategories` uses Levenshtein edit distance (≤ 2) for "did you mean" suggestions. `selectorToOverpassFilter` renders selectors as Overpass tag filters with `=` or `~` (regex) operators.

**`providers/overpass.ts`.** Builds Overpass QL in one of two forms: a broad default query over 10 selector families, or a **targeted** query when selectors are supplied. Then it:
- posts the query form-encoded as `data=`;
- detects Overpass errors that arrive in a `remark` field with HTTP 200;
- classifies each element and attaches `completeness` and `lastVerified`;
- runs `dedupePois`.

Way and relation geometry uses `out center tags meta` (the bounding-box center). True polygon centroids are still an open roadmap item.

**`providers/nominatim.ts`.** `jsonv2` search and reverse geocoding. Each instance has its own `RateLimiter`, defaulting to 1 req/s (OSM policy), and sends a contact `User-Agent`. Nominatim's `importance` value is passed through in `raw` so `disambiguate.ts` can judge ambiguity.

**`http.ts`.** A dependency-free `fetch` wrapper:
- `RateLimiter` serializes calls through a promise chain spaced `minIntervalMs` apart.
- `requestJson` retries with exponential backoff, only on retryable failures (429, 5xx, timeout, network; see `isRetryable`).
- An optional `RequestCache` keys responses by method + URL + body.
- An external `AbortSignal` is merged with an internal `AbortSignal.timeout`.

**`hours.ts`.** A hand-written `opening_hours` evaluator for the common grammar: weekday ranges and lists, multiple and overnight time ranges, `24/7`, `off`/`closed`, and `;` rule overrides. It returns `unknown` rather than guess when the string contains anything it can't evaluate: `sunrise`/`sunset`, months, years, `week` selectors, or open-ended `08:00+`.

Public- and school-holiday rules (`PH`/`SH`) are **skipped**, since there is no holiday calendar. A string like `Mo-Fr 09:00-17:00; PH off` is therefore evaluated from its regular rules. The v1.0.1 fix covers strings that contain *only* holiday rules (e.g. `PH off`). These now return `unknown` instead of `closed`, because no regular schedule backs a state. (The June version of this doc said PH/SH rules return `unknown`, which was imprecise.)

The evaluator also computes `nextChange`, the next open/closed transition, searching up to 8 days ahead. Times are read as local wall-clock time. Callers must pass a `when` already shifted into the POI's timezone.

**`routing.ts` + adapters.**
- `HaversineRoutingProvider` is the key-free fallback. Distance is haversine, duration uses per-mode speeds (`walk 1.4`, `bike 4.2`, `drive 11.1` m/s), and the isochrone is a 48-point circle polygon.
- `ValhallaRoutingProvider` defaults to `valhalla1.openstreetmap.de` (FOSSGIS, key-free). It supports matrices (`/sources_to_targets`) and real isochrone polygons (`/isochrone`), mapping modes to `pedestrian`/`bicycle`/`auto`.
- `OsrmRoutingProvider` computes matrices via the Table service. The public demo is car-only and has no isochrone.

`reachable.ts` decides membership by point-in-polygon when an isochrone is available, and otherwise by a matrix budget threshold. A road-network isochrone can contain a POI whose matrix travel time still exceeds the budget. Since v1.0.1 the per-POI `score` is clamped to `[0, 1]` (it used to go negative in that case).

**`errands.ts` — the most novel feature.** Solves the **Generalized TSP**: visit one POI per category with the least total travel. It fetches the nearest K candidates per category (default `K = 5`, within a 3 km search radius) and builds a full cost matrix. It then runs an exact **grouped Held-Karp DP** over `2^groupCount` bitmask states, capped at `MAX_CATEGORIES = 12`, with an optional fixed end point. The visit order is rebuilt from a parent table.

The result is optimal over the shortlisted candidates. It is not optimal over every POI in the area, because the top-K trim is a heuristic. Categories with no candidate are reported as `missing`, not faked.

**`walkability.ts` / `gaps.ts` / `compare.ts`.**
- `walkSubScore` is a transparent linear distance decay: full credit at ≤ 400 m and zero at ≥ 2,400 m by default. Sub-scores are weighted across a tunable daily-needs basket to give a 0–100 score. The `confidence` is 0.7·coverage + 0.3·average completeness.
- `detectGaps` flags categories with no match within a distance threshold.
- `compareLocations` is pure composition. It runs `walkabilityScore` once per location (sequentially, to respect OSM rate limits), ranks the locations, and reports a winner per dimension.

**`ranking.ts` / `filters.ts` — scorers.** The default proximity scorer and `accessibleScorer` both divide by `Math.max(radiusMeters, 1)` (v1.0.1). Before this guard, a POI at the origin with `radiusMeters: 0` produced `0/0 = NaN`, which broke the sort. `accessibleScorer` puts POIs in tiers: `wheelchair=yes` at 0.66, `limited` at 0.33, others at 0. Proximity now adds at most 0.32, which is smaller than the 0.33 gap between tiers. That keeps the tiers strictly separate, so a step-free POI at the radius edge always outranks a `limited` one at the origin. With the old 0.33 span, those two tied.

**CLI (`cli/src/index.ts` + `render.ts`).** Commander with 9 commands.
- `parseFilters` turns repeatable `--filter key=value` options (and bare `key`) into `FacetFilters`.
- `parseMode`, `parseWithin`, and `parseWeights` handle aliases and budgets.
- `render.ts` produces colored terminal output via picocolors, including open/closed and ♿ accessibility marks and score breakdowns. Since v1.0.1 the geocode ambiguity banner reads "distinct places share this name; showing N candidates". It previously claimed "N distinct places".
- `--format geojson|csv` writes data to stdout and the ODbL notice to stderr, so the data stays pipeable.

**MCP (`mcp/src/index.ts` + `payload.ts`).** Registers **8 tools** with zod input schemas on an `McpServer` over stdio. The travel-time and reachable tools use the key-free Valhalla engine. `payload.ts` flattens each core result into a slim JSON shape for agents. `find_nearby_amenities` has a `concise` mode (which implies `explain`) to save tokens. Errors are returned as `isError` text content rather than thrown.

## Core features & public API/commands

Unchanged since v1.0.0. The 1.0.1 release was fixes only.

| Capability | CLI command | Library function | MCP tool |
| --- | --- | --- | --- |
| Nearby amenities, ranked by distance | `near` | `findNearbyAmenities` | `find_nearby_amenities` |
| …by travel time (walk/bike/drive) | `near --by travel-time` | `findNearbyAmenities({ rankBy })` | `find_nearby_amenities` |
| Facet filters + accessibility-first | `near --filter … --accessible` | `findNearbyAmenities({ filters, accessible })` | `find_nearby_amenities` |
| Open now / open at | `near --open-now` / `--open-at` | `findNearbyAmenities({ open })`, `isOpenAt` | `find_nearby_amenities` |
| Geocode with disambiguation | `geocode` | `disambiguateLocation` | `geocode` |
| Gap detection ("what's missing") | `gaps` | `detectGaps` | `detect_amenity_gaps` |
| Walkability score (0–100 + breakdown) | `score` | `walkabilityScore` | `walkability_score` |
| Compare N locations | `compare` | `compareLocations` | `compare_locations` |
| Reachability in N min (isochrone) | `reachable` | `reachableAmenities` | `reachable_amenities` |
| Errand planner (Generalized TSP) | `errands` | `planErrands` | `plan_errands` |
| Export GeoJSON / CSV | `near --format …` | `toGeoJSON`, `toCSV` | — |
| Snapshot an area, query offline | `snapshot` / `near --dataset` | `snapshotArea`, `DatasetPlacesProvider` | — |
| Bulk scoring → CSV | `bulk` | `walkabilityScore` loop | — |
| List the category vocabulary | — | `categoryVocabulary` | `list_categories` |

The 16 normalized categories (`CATEGORIES` in `types.ts`): food, grocery, shopping, healthcare, education, finance, transport, fuel, parking, accommodation, leisure, tourism, worship, public_service, utility, other.

## Map/geo data sources & integrations

All sources are open, key-free, and based on OpenStreetMap:

- **Geocoding:** Nominatim (`nominatim.openstreetmap.org`), `jsonv2`, default rate limit 1 req/s.
- **POIs:** Overpass API (`overpass-api.de/api/interpreter`), Overpass QL `around:` queries.
- **Routing/isochrones:** the Valhalla FOSSGIS instance (`valhalla1.openstreetmap.de`) by default. OSRM (`router.project-osrm.org`, car-only demo) is an alternative. `HaversineRoutingProvider` is the always-available, network-free fallback. proximap *consumes* isochrones from Valhalla rather than computing them. Without Valhalla, the "isochrone" is a straight-line circle.
- **Data license:** OSM data is under **ODbL**, which (unlike commercial map APIs) permits storing and exporting results. `export.ts` and `snapshot.ts` emit the `ODBL_ATTRIBUTION` notice, and the CLI prints it to stderr.

Every endpoint is configurable (`endpoint`, `userAgent`, `timeoutMs`, `minIntervalMs`, `retries`, `cache`), so users can self-host or bring their own backend. There is no Leaflet/MapLibre/Mapbox GL/tile-rendering layer anywhere in the codebase.

## Tech stack & build tooling

- **Language:** TypeScript ^6 (strict, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, Bundler resolution). `tsconfig.base.json` sets `ignoreDeprecations: "6.0"` to work around a TS6 baseUrl deprecation that tsup's d.ts step injects.
- **Build:** **tsup** per package. Core emits dual **ESM + CJS** plus `.d.ts`/`.d.cts` (`format: ['esm','cjs']`, `dts: true`, sourcemaps, target es2022). CLI and MCP build ESM executables.
- **Tests:** **vitest** (run from the root with `vitest run`), node environment.
- **Lint/format:** Prettier (`format` / `format:check`); `.editorconfig`, `.prettierrc`, `.nvmrc`.
- **Lockfiles:** a root `package-lock.json` (~128 KB) and a `packages/cli/node_modules` exist. Both were excluded from the counts.
- **CI:** `.github/workflows/ci.yml` runs `npm ci → build → typecheck → format:check → test` on Node **20 and 24** on every push to main and every PR. `.github/workflows/release.yml` runs on `v*` tags. It runs `npm run check`, publishes all three packages to npm with **provenance (OIDC)** using an `NPM_TOKEN` secret, and creates a GitHub release. Both the v1.0.0 and v1.0.1 Release runs and the matching CI runs on `main` succeeded.

## Build / dev / test (actual commands)

```bash
npm install        # install workspace deps (Node ≥ 20)
npm run build      # tsup: build @proximap/core, then @proximap/cli + @proximap/mcp
npm run typecheck  # tsc --noEmit across workspaces
npm test           # vitest run
npm run test:watch # vitest watch
npm run format     # prettier --write .
npm run check      # build + typecheck + format:check + test — the per-commit gate
```

Example CLI usage (from README / `examples/demo.ps1`):

```bash
proximap near "Eiffel Tower, Paris" --radius 1000 --limit 20
proximap near "Kreuzberg, Berlin" -c food --by travel-time --filter diet=vegan --filter takeaway
proximap score "Brandenburg Gate, Berlin"
proximap errands "Alexanderplatz, Berlin" -c pharmacy -c atm -c grocery
proximap snapshot "Montmartre, Paris" --out montmartre.json
proximap near "48.8867,2.3431" -c cafe --dataset montmartre.json   # fully offline
```

## Tests

**22 `.test.ts` files, 139 test cases (1,956 LOC)**, colocated with the source. In June the count was 132 cases in 1,867 LOC.

Coverage is concentrated in core, which has **20 test files**: `categories`, `taxonomy`, `compare`, `disambiguate`, `errands`, `export`, `filters`, `gaps`, `geo`, `hours` (18 cases, the largest), `http`, `nearby`, `quality`, `ranking`, `reachable`, `routing`, `snapshot`, and `walkability`, plus the provider tests (`nominatim`, `overpass`). The CLI has one test file (`render.test.ts`, 7 cases) and the MCP package one (`payload.test.ts`, 3 cases). The payload and rendering shapes are tested, and the engine logic heavily so.

The seven new cases are **regression tests for the v1.0.1 fixes**, one or two per fix:
- reachable score clamp
- holiday-only hours
- `radiusMeters: 0` NaN guard, in both `ranking` and `filters`
- strict accessibility tier separation
- `formatDistance` at the 1 km boundary
- ambiguity banner wording

`vitest.config.ts` sets `passWithNoTests: true`. The CHANGELOG says everything was "verified against live OpenStreetMap," but the committed tests are unit tests over fixtures and pure functions, not live-network tests.

## Status, completeness & notable gaps

- **Status: v1.0.1, published and unchanged since 2026-06-18.** Checked against the live npm registry on 2026-10-12:
  - `@proximap/core`, `@proximap/cli`, and `@proximap/mcp` each have versions `1.0.0` and `1.0.1`, with `dist-tags.latest = 1.0.1`. All three 1.0.1 tarballs were published 2026-06-19 UTC.
  - Each carries npm provenance attestations. Their `gitHead` is `0d4ba89`, which matches the repo HEAD.
  - The published `@proximap/core` declares no dependencies, matching the "zero runtime deps" claim.
  - GitHub shows two releases, `v1.0.0` and `v1.0.1` (Latest).
- **What changed since June: one audit-driven patch release.** The 1.0.1 CHANGELOG describes "correctness fixes from a multi-agent audit sweep (each verified with a regression test). No API changes." The six fixes are listed in the walkthrough above (`reachable`, `hours`, `ranking`/`filters`, `geo`, `cli`).
- **Development has stopped since then.** All 36 commits fall within 2026-06-16 → 2026-06-18. There has been no activity for roughly four months. The full researched backlog (proposals 01–12) is implemented, and the three surfaces share one engine.
- **Still planned / not implemented (no change since June):**
  - `apps/web` (Web UI) is still a placeholder README only.
  - `python/` (PyPI port) has a README and an empty, untracked `src/proximap/` directory, but **no `.py` files**.
  - The Claude Code plugin wrapper around the MCP server is on the roadmap but not built.
  - "More providers (Google Places, Mapbox, Foursquare)" is also still unbuilt.
- **Acknowledged limitations (all five ROADMAP "v1 follow-ups" are still unchecked):**
  - Way/relation centroids use Overpass `out center` rather than true polygon centroids.
  - `opening_hours` has no timezone or holiday-calendar support. 1.0.1 only stopped holiday-only strings from reading as `closed`.
  - MCP lacks `outputSchema`/`structuredContent`/elicitation (no references in `packages/mcp/src`).
  - Facet filtering happens after the fetch and is not pushed down to Overpass.
  - Parquet export and bulk over `near`/`gaps` are pending.
- **Operational caveat:** the defaults hit shared public OSM instances with limits of about 1 req/s, so heavy use needs self-hosting. Every endpoint is configurable, so this is supported.

## README vs. code

The README is still unusually accurate. Spot-checks against v1.0.1:

- "Ships as a library, a CLI, and an MCP server": **true** (three packages, all built and published at 1.0.1).
- "Dependency-free core … uses the platform `fetch`": **true**. `@proximap/core` declares no `dependencies` (in the repo and in the published 1.0.1 manifest), and `http.ts` uses the global `fetch`.
- "16 categories": **true** (`CATEGORIES` has exactly 16 members).
- CLI command list and MCP tool list: **match the code** (9 commander `.command()` calls; 8 `registerTool` calls).
- "Valhalla FOSSGIS instance … no keys" / "Nominatim + Overpass": **true** (default endpoints in the provider constructors).
- "Status: v1.0 — published on npm. The public API is stable per SemVer": **true**. 1.0.1 is a patch with no API changes, consistent with SemVer.
- CHANGELOG 1.0.1, "each verified with a regression test": **true**. Every one of the six fixes has a matching new test (seven new cases).

Mismatches and nits:

1. **`python/README.md` says "The package name is reserved on PyPI."** It **can't be verified.** On 2026-10-12, `https://pypi.org/pypi/proximap/json` returns **404**, so no public `proximap` project exists. A "pending trusted publisher" would not be visible publicly, but nothing on PyPI backs this claim. (New in this refresh.)
2. `DEFAULT_USER_AGENT` in `http.ts` still reads `proximap/0.1`, while the packages are now at **1.0.1**. This is a cosmetic version-string lag, not a functional issue. Version strings are also hard-coded in three places (`core/src/index.ts`, `cli/src/index.ts`, `mcp/src/index.ts`) plus four `package.json` files, all bumped by hand.
3. The README's MCP table lists "List the category vocabulary" (`list_categories`) with "—" under CLI. The code confirms there is no CLI command for it (`categoryVocabulary` is library/MCP-only), so the table is consistent.
4. README, ARCHITECTURE, ROADMAP, and CHANGELOG explicitly describe `apps/web` and `python/` as "planned"/"not yet implemented," and the empty directories confirm it. Apart from the PyPI-reservation line, nothing is overclaimed.
5. The working directory is named "MapsPlugin," but nothing in the code is a plugin or a map renderer. The name is misleading about what the project is.

---

*Doc generated 2026-06-16, refreshed 2026-10-12 by reading the source at C:\Users\ameya\Documents\MapsPlugin. Source of truth: https://github.com/AmeyaBorkar/proximap*
