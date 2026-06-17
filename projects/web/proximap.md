# proximap

> A key-free geospatial toolkit for **places, proximity, and amenities** built on OpenStreetMap data. It is **not** a map-rendering plugin (no Leaflet/MapLibre/Obsidian/Figma host) — it is a TypeScript engine that geocodes a place, fetches points of interest from OSM, and answers higher-order questions: rank nearby amenities by distance or real travel time, score walkability, detect amenity gaps, list what's reachable in N minutes, plan a shortest multi-stop errand, and compare neighbourhoods. Shipped as three published npm packages — a library (`@proximap/core`), a CLI (`@proximap/cli`), and an MCP server (`@proximap/mcp`) for AI agents.

| | |
| --- | --- |
| **Repository** | https://github.com/AmeyaBorkar/proximap |
| **Visibility** | Public |
| **Category** | web |
| **Primary language(s)** | TypeScript (100% of source; zero runtime deps in core) |
| **Local path** | `C:\Users\ameya\Documents\MapsPlugin` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 5,190 source TS + 1,867 test TS = **7,057 LOC of TypeScript**; plus ~1,388 lines of Markdown docs |
| **Source files (computed)** | 33 non-test `.ts` modules (excludes the 3 `tsup`/`vitest` configs) + 22 `.test.ts` files = 55 TS files; 132 test cases |
| **Package / host platform** | npm-workspaces monorepo, Node ≥ 20. Three published packages: `@proximap/core` (library), `@proximap/cli` (bin `proximap`), `@proximap/mcp` (bin `proximap-mcp`, stdio MCP server). Root pkg `proximap-monorepo` is private. |
| **Key dependencies** | core: **none at runtime** (platform `fetch`). cli: `commander` ^15, `picocolors` ^1. mcp: `@modelcontextprotocol/sdk` ^1.29, `zod` ^4. dev: `tsup`, `vitest`, `typescript` ^6, `prettier`. |
| **License** | MIT © Ameya Borkar |
| **Last commit** | `2a849d4` — "ci: tag-triggered npm release workflow with provenance" (2026-06-16). First commit `5146e59` same day; 29 commits total. |

---

## What it actually is (be concrete)

Despite the local working-directory name "MapsPlugin," **proximap is not a plugin for any map host.** There is no `manifest.json`, no Obsidian/Figma manifest, no Leaflet/MapLibre/Mapbox GL dependency, and nothing renders tiles. It is a **headless geospatial reasoning engine** plus three delivery surfaces.

The core abstraction is three small provider interfaces (`types.ts`):

- **`GeocodingProvider`** — `geocode(query)` / optional `reverse(latlng)`. Default impl: `NominatimGeocoder` (OSM Nominatim).
- **`PlacesProvider`** — `findNearby(center, { radiusMeters, selectors })`. Default impl: `OverpassPlacesProvider` (OSM Overpass API); also `DatasetPlacesProvider` for offline snapshots.
- **`RoutingProvider`** — `matrix(origin, targets, mode)` + optional `isochrone(...)`. Default impl: `HaversineRoutingProvider` (straight-line, no network); real engines `ValhallaRoutingProvider` and `OsrmRoutingProvider`.

The headline entry point is `findNearbyAmenities(query, options)` in `packages/core/src/nearby.ts`. It resolves an origin (place name, `"lat,lng"` string, or `LatLng`), fetches POIs, classifies each into one of **16 normalized categories**, applies facet/open-now filters, then ranks by distance (haversine) or travel time (routing matrix with haversine fallback). Everything else (`walkabilityScore`, `detectGaps`, `reachableAmenities`, `compareLocations`, `planErrands`, `snapshotArea`) **composes** that same pipeline and the same providers.

Design stance, evident in the code and docs: **key-free defaults** (OSM Nominatim + Overpass + the FOSSGIS Valhalla instance, no billing), **honesty about data** (absence is framed as "not found in OSM," scores carry a `confidence`, ambiguous names return candidates not a guess), and **agent-native output** (deterministic ordering via a stable `id` tie-break, an `explain` mode, and a `concise` MCP payload).

## Architecture & how it's structured

npm-workspaces monorepo (`workspaces: ["packages/*"]`), Node ≥ 20, ESM-first. Annotated tree (node_modules/dist omitted):

```
MapsPlugin/
├─ package.json              root: "proximap-monorepo" (private), build/test/check scripts
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
└─ python/README.md          PyPI port — PLANNED, not implemented (no .py files exist)
```

**Entry points:** library → `packages/core/src/index.ts` (re-exports everything; `VERSION = '1.0.0'`). CLI → `packages/cli/src/index.ts` (commander `program.parseAsync()`). MCP → `packages/mcp/src/index.ts` (`McpServer` over `StdioServerTransport`).

## Code walkthrough (the important parts)

**`nearby.ts` — `findNearbyAmenities()`.** The pipeline: resolve NL category terms to selectors (`resolveSelectors` → throws with typo suggestions on unknown terms); `resolveOrigin`; `places.findNearby`; post-filter to matching selectors; apply compiled facet predicates; drop confirmed-closed POIs when `open` is set (keeping open + unknown, never silently dropping unknown-hours places); rank. Travel-time ranking (`rankByTravelTime`) trims to the nearest `TRAVEL_MATRIX_CAP = 80` candidates by haversine first (to bound matrix size and public-engine cost), calls `routing.matrix`, and **falls back to haversine if the engine errors** (setting `routing.fellBack`). `explain` attaches a deterministic `rankingReason` like "closest open cafe, 240 m" with proper English ordinals.

**`categories.ts` / `taxonomy.ts` — the two-way vocabulary.** `categorize(tags)` (reading direction) maps OSM tags → one of 16 categories in priority order (`amenity` → `shop` → `tourism` → `healthcare` → `leisure` → `railway`/`public_transport`/`highway`/`aeroway` → `office`), preserving the raw value as `kind`. `taxonomy.ts` (query direction) holds a `TERMS` table mapping human words ("chemist", "petrol", "cashpoint") and synonyms to OSM `CategorySelector[]`, deduped, and rolled up to a category. `suggestCategories` uses Levenshtein edit distance (≤ 2) for "did you mean" suggestions. `selectorToOverpassFilter` renders selectors as Overpass tag filters with `=` or `~` (regex) operators.

**`providers/overpass.ts`.** Builds Overpass QL: a broad default query over 10 selector families, or a **targeted** query when selectors are supplied. Posts form-encoded `data=`, detects Overpass errors that arrive in a `remark` with HTTP 200, classifies each element, attaches `completeness`/`lastVerified`, and runs `dedupePois`. Way/relation geometry uses `out center` (the centroid) — note the roadmap flags true polygon centroids as a follow-up.

**`providers/nominatim.ts`.** `jsonv2` search + reverse, with a per-instance `RateLimiter` defaulting to 1 req/s (OSM policy) and a contact `User-Agent`. Carries Nominatim's `importance` through in `raw` so `disambiguate.ts` can judge ambiguity.

**`http.ts`.** Dependency-free `fetch` wrapper. `RateLimiter` serializes calls via a promise chain spaced `minIntervalMs` apart. `requestJson` does retry with exponential backoff (only on 429/5xx/timeout/network — `isRetryable`), optional `RequestCache` keyed by method+url+body, and merges an external `AbortSignal` with an internal `AbortSignal.timeout`.

**`hours.ts`.** A hand-rolled `opening_hours` evaluator for the common grammar: weekday ranges/lists, multiple/overnight time ranges, `24/7`, `off`/`closed`, and `;` rule overrides. Anything it can't justify (`sunrise`/`sunset`, months, years, open-ended `08:00+`, `PH`/`SH` holidays) returns `unknown` — never a guess. Also computes `nextChange` (next open/closed transition, searching up to 8 days). Times are read as local wall-clock; the roadmap flags timezone correctness as a follow-up.

**`routing.ts` + adapters.** `HaversineRoutingProvider` is the key-free floor: distance via haversine, duration via per-mode speeds (`walk 1.4`, `bike 4.2`, `drive 11.1` m/s), isochrone as a 48-point circle polygon. `ValhallaRoutingProvider` (default endpoint `valhalla1.openstreetmap.de`, FOSSGIS, key-free) supports matrices (`/sources_to_targets`) and real isochrone polygons (`/isochrone`), mapping modes to `pedestrian`/`bicycle`/`auto`. `OsrmRoutingProvider` does matrices via the Table service (public demo is car-only; no isochrone).

**`errands.ts` — the most novel feature.** Solves the **Generalized TSP** ("visit one POI per category, minimize total travel"): fetch nearest K candidates per category, build a full cost matrix, then an exact **grouped Held-Karp DP** over `2^groupCount` bitmask states (capped at `MAX_CATEGORIES = 12`), with an optional fixed end point. Reconstructs the visit order from a parent table. Categories with no candidate are reported as `missing`, not faked.

**`walkability.ts` / `gaps.ts` / `compare.ts`.** `walkSubScore` is a transparent linear distance-decay (full credit ≤ 400 m, zero ≥ 2400 m by default), weighted across a tunable daily-needs basket → 0–100 plus a `confidence` (0.7·coverage + 0.3·avg completeness). `detectGaps` flags categories with no match within a threshold. `compareLocations` is pure composition over `walkabilityScore` run per location (sequentially, to keep OSM rate limiters polite), then ranked with a per-dimension winner.

**CLI (`cli/src/index.ts` + `render.ts`).** Commander with 9 commands. `parseFilters` turns repeatable `--filter key=value` (and bare `key`) into `FacetFilters`; `parseMode`/`parseWithin`/`parseWeights` handle aliases and budgets. `render.ts` produces colored terminal output via picocolors (open/closed and ♿ accessibility marks, score breakdowns). `--format geojson|csv` writes data to stdout and the ODbL notice to stderr so the data stays pipeable.

**MCP (`mcp/src/index.ts` + `payload.ts`).** Registers **8 tools** with zod input schemas on an `McpServer` over stdio. Travel-time/reachable tools wire in the key-free Valhalla engine. `payload.ts` flattens each core result into a slim, agent-friendly JSON shape; `find_nearby_amenities` has a `concise` mode (implies `explain`) for token economy. Errors are returned as `isError` text content rather than thrown.

## Core features & public API/commands

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

All open, key-free, OpenStreetMap-based:

- **Geocoding** — Nominatim (`nominatim.openstreetmap.org`), `jsonv2`, default 1 req/s rate limit.
- **POIs** — Overpass API (`overpass-api.de/api/interpreter`), Overpass QL `around:` queries.
- **Routing/isochrones** — Valhalla FOSSGIS instance (`valhalla1.openstreetmap.de`) by default; OSRM (`router.project-osrm.org`, car-only demo) as an alternative; `HaversineRoutingProvider` as the always-available, network-free fallback.
- **Data license** — OSM under **ODbL**, which (unlike commercial map APIs) permits storing/exporting results; the `ODBL_ATTRIBUTION` notice is emitted by `export.ts` and `snapshot.ts` and printed to stderr by the CLI.

Every endpoint is overridable (`endpoint`, `userAgent`, `timeoutMs`, `minIntervalMs`, `retries`, `cache`) so users can self-host or bring their own backend. There is no Leaflet/MapLibre/Mapbox GL/tile-rendering layer anywhere in the codebase.

## Tech stack & build tooling

- **Language:** TypeScript ^6 (strict, `noUncheckedIndexedAccess`, `verbatimModuleSyntax`, Bundler resolution). `tsconfig.base.json` sets `ignoreDeprecations: "6.0"` to work around a TS6 baseUrl deprecation injected by tsup's d.ts step.
- **Build:** **tsup** per package. Core emits dual **ESM + CJS** plus `.d.ts`/`.d.cts` (`format: ['esm','cjs']`, `dts: true`, sourcemaps, target es2022). CLI and MCP build ESM executables.
- **Tests:** **vitest** (run from root, `vitest run`), node environment.
- **Lint/format:** Prettier (`format` / `format:check`); `.editorconfig`, `.prettierrc`, `.nvmrc`.
- **Lockfiles:** root `package-lock.json` (~128 KB) and a `packages/cli/node_modules` exist; not read per task instructions.
- **CI:** `.github/workflows/ci.yml` runs `npm ci → build → typecheck → format:check → test` on Node **20 and 24** on every push to main and PR. `.github/workflows/release.yml` publishes all three packages to npm with **provenance (OIDC)** and cuts a GitHub release on `v*` tags (needs `NPM_TOKEN`).

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

**22 `.test.ts` files, 132 test cases (1,867 LOC)**, colocated with source. Coverage is concentrated in core (**20 test files**) — `categories`, `taxonomy`, `compare`, `disambiguate`, `errands`, `export`, `filters`, `gaps`, `geo`, `hours`, `http`, `nearby`, `quality`, `ranking`, `reachable`, `routing`, `snapshot`, `walkability`, plus provider tests (`nominatim`, `overpass`). The CLI has 1 test (`render.test.ts`) and the MCP package 1 (`payload.test.ts`) — i.e. the payload/rendering shapes are tested, the engine logic heavily so. `vitest.config.ts` uses `passWithNoTests: true`. The CHANGELOG states everything was "verified against live OpenStreetMap," but the committed tests are unit tests over fixtures/pure functions, not live-network tests.

## Status, completeness & notable gaps

- **Status: v1.0.0, genuinely published.** Verified against the live npm registry: `@proximap/core`, `@proximap/cli`, and `@proximap/mcp` all resolve with `dist-tags.latest = 1.0.0`. The README's "published on npm" claim is accurate.
- **Young but complete-feeling.** All 29 commits land on 2026-06-16 (a single intensive day). The full researched backlog (proposals 01–12) is implemented; the three surfaces share one engine; CI is green on two Node versions.
- **Planned / not implemented:** `apps/web` (Web UI) is a placeholder README only; `python/` (PyPI port) has **no `.py` files** — name reserved, layout sketched. A Claude Code plugin wrapper around the MCP server is roadmapped, not built.
- **Acknowledged limitations (CHANGELOG + ROADMAP):** way/relation centroids use Overpass `out center` rather than true polygon centroids; `opening_hours` ignores timezones and holidays (local wall-clock only); MCP lacks `outputSchema`/`structuredContent`/elicitation; facet filtering is post-fetch (not pushed down to Overpass); Parquet export and bulk over `near`/`gaps` are pending. These are flagged as non-breaking follow-ups.
- **Operational caveat:** defaults hit shared public OSM instances with ~1 req/s limits; heavy use needs self-hosting (the code makes every endpoint overridable, so this is supported).

## README vs. code

The README is unusually accurate. Spot-checks:

- "Ships as a library, a CLI, and an MCP server" — **true** (three packages, all built and published).
- "Dependency-free core … uses the platform `fetch`" — **true**: `@proximap/core` declares no `dependencies`; `http.ts` uses global `fetch`.
- "16 categories" — **true** (`CATEGORIES` has exactly 16 members).
- CLI command list and MCP tool list — **match the code** (9 commander commands; 8 `registerTool` calls).
- "Valhalla FOSSGIS instance … no keys" / "Nominatim + Overpass" — **true** (default endpoints in the provider constructors).
- "Published on npm, public API stable per SemVer" — **true** per the live registry check.

Minor nits, not contradictions: (1) The README's MCP table lists "List the category vocabulary" with `list_categories` but a "—" under CLI; the code confirms there is no CLI command for it (`categoryVocabulary` is library/MCP-only) — consistent. (2) `DEFAULT_USER_AGENT` in `http.ts` still reads `proximap/0.1` while the packages are v1.0.0 — a cosmetic version-string lag, not a functional issue. (3) README, ARCHITECTURE, ROADMAP, and CHANGELOG describe planned `apps/web` and `python/` as explicitly "planned"/"not yet implemented," which the empty directories confirm — no overclaiming. (4) The working directory is named "MapsPlugin," but nothing in the code is a plugin or a map renderer; the name is misleading relative to what the project is.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\MapsPlugin. Source of truth: https://github.com/AmeyaBorkar/proximap*
