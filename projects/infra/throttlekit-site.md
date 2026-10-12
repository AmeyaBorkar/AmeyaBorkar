# throttlekit-site

> The static marketing/docs site for **ThrottleKit** — the rate limiter that "meters what your LLM spends and proves what your fleet admits." Built with Astro 6, it ships essentially zero client JS apart from one inline canvas hero (a 120-frame scroll-scrubbed "feather" animation) and, since late June, a light/dark theme toggle. It renders the library's design docs and editorial blog from on-repo markdown content collections, and hand-codes its entire design system in a single 590-line CSS file (no Tailwind): monochrome with a pale-gold accent, plus a "paper" light theme. Canonical origin is `https://throttlekit.in`; it builds to a static `dist/` with no server runtime.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/throttlekit-site |
| **Visibility** | Private |
| **Category** | infra |
| **Primary language(s)** | Astro (`.astro` components/pages) + TypeScript (data/config) + hand-written CSS; Markdown content |
| **Local path** | `C:\Users\ameya\Documents\GreenfeildProject\research\website\site`. This is a full clone with its own `.git` (remote `AmeyaBorkar/throttlekit-site`), nested inside the core repo's working tree and excluded by the core's `.gitignore`. The June doc used a shallow cache at `C:\Users\ameya\.repo-cache\throttlekit-site`, which no longer exists. |
| **Default branch** | `main` |
| **Lines of code** | ~6,530 lines of code/content across `src/` + `scripts/` (Astro 1,979 · Markdown content 3,304 · CSS 590 · TS 553 · MJS 81 · JS 23), up ~480 from June. **Assets dominate the repo by size:** ~5.0 MB in `public/` (2.5 MB of 120 WebP hero frames + 2.4 MB `feather.mp4` + a 44 KB OG image), unchanged. |
| **Source files** | 20 `.astro`, 6 `.ts`, 2 `.mjs`, 1 `.js`, 1 `.css`, 24 `.md` (18 design docs + **6** blog posts, was 4), plus `public/` (125 files: 120 frames, video, favicon `.svg`, touch icon, OG image, `robots.txt`). 186 tracked files in total. |
| **Key dependencies** | `astro ^6.4.2`, `@astrojs/sitemap ^3.7.3`, `@astrojs/rss ^4.0.18`. That's the **entire** dependency tree — no UI framework, no Tailwind, no CSS toolchain. (Unchanged.) |
| **License** | MIT per `src/config.ts` (`SITE.license = "MIT"`), but **no LICENSE file exists in this repo** — the footer/about links point at `…/throttlekit/blob/main/LICENSE` in the core library repo. |
| **Last commit** | `0fa92ed` — "content: sync 1.6.1/1.7.0 + server 0.4.4 + py 0.5.1, resync benchmarks, cut em-dash overuse" · 2026-07-02 (matches GitHub `main`, last push 2026-07-02). 43 commits in full history, 5 since the June doc. |

## What it actually is

A **static Astro marketing + documentation site** for the ThrottleKit rate-limiter project. It is content-heavy and code-light: the bulk of the "lines" are the 18 design-doc markdown files (2,778 lines) and 6 blog posts (526 lines), which are rendered through Astro **content collections**. The actual application logic is small — config/data modules, three layouts, three components, a rehype link-rewriter plugin, and ~13 routes.

Confirmed from the source, not the README:

- **Astro 6.4.2**, output is the default **static** build (`astro build → dist/`). The only integration wired in `astro.config.mjs` is `@astrojs/sitemap`. RSS is served by a route (`rss.xml.js`) using `@astrojs/rss`, not an integration.
- **No client framework.** There is no React/Vue/Svelte/Solid integration and no Astro `client:*` island anywhere. The "hydrated island" the README mentions is really **inline vanilla `<script>`** blocks in `index.astro`, `Background.astro`, `Nav.astro`, and `Base.astro` — plain DOM JS, not an Astro framework island.
- **No Tailwind, no PostCSS config.** Styling is one hand-authored stylesheet, `src/styles/global.css` (590 lines), plus a few scoped `<style>` blocks. `grep` for tailwind in the lockfile returns 0.
- **No Vercel/Netlify adapter or config file.** No `vercel.json`, `netlify.toml`, or adapter in `astro.config.mjs`. It's a pure static `dist/` that the README says is "hosted on Vercel," but nothing in the repo configures that.

**What changed since June (5 commits, 2026-06-24 → 07-02):**
- **Light/dark theme** (`ff881bc`, `56b36e3`, `72b7537`). This adds a sun/moon toggle in the nav, a pre-paint theme resolver in `Base.astro`, roughly 97 lines of `html[data-theme="light"]` overrides, and a fix that keeps the cinematic home hero light-on-dark in light mode.
- **Content syncs for every release since 1.5.x** (`cb1b1c3`, `0fa92ed`). `changelog.ts` gained throttlekit 1.6.0/1.6.1/1.7.0, server 0.4.3/0.4.4 and py 0.5.1. Two new blog posts cover the 1.6.0 audit sweep and the 1.6.1 → 1.7.0 perf-then-hardening releases. The release-history post was extended, `data/bench.ts` was resynced to the current `BENCH.md`, and the 1.7.0 hardening invariants were written into the design docs.
- **A site-wide copy pass** that removed most em-dashes from the prose. The commit message explains that the writing "read noticeably AI-generated."

## Where it sits in the ThrottleKit family

This is one of three sibling repos. The site references all three explicitly (`src/config.ts`):

| Repo | Role | How this site links it |
|---|---|---|
| `throttlekit` (TS core) | The library — algorithms, stores, adapters, the two engines (TALE/GALE), gRPC server + Lens | `SITE.gh`, `SITE.npm` (npm `throttlekit`), the canonical home for design docs and `LICENSE`/`BENCH.md`/`STABILITY.md`/`SCOREBOARD.md` |
| `throttlekit-py` (Python client) | The reference Python client | `SITE.pyGh`, `SITE.pypi` (PyPI `throttlekit-py`); the cost-axis (`debit`) code samples on the home + `/tale` pages target it |
| **`throttlekit-site`** (this repo, Astro) | The marketing + docs front-end | — |

The site treats the **core library repo as the source of truth**. Three mechanisms connect them, and all three are manual:

- `scripts/sync-design.mjs` (run manually via `npm run sync:design`) copies `docs/design/NN-*.md` from the monorepo (`../../../../docs/design`) into `src/content/design/`. The relative path works because this clone lives at `GreenfeildProject/research/website/site`. Vercel builds only from the committed copies — it can't read the monorepo. **Since the July 2 commit the copies are no longer verbatim.** All 18 site design docs now differ from the core's `docs/design/*.md` (compared with line endings normalized). The site copies had em-dashes stripped (e.g. `01-core-model.md`: 24 em-dashes in the core, 0 on the site), and some carry site-only edits, such as a longer decision-finiteness section in `02-algorithms.md`. Re-running `sync:design` would overwrite those edits.
- `src/data/bench.ts` is a **hand-maintained mirror** of the library's `BENCH.md` (the comment says "Vercel builds only from this repo and can't read the monorepo's BENCH.md… When BENCH.md changes, update here"). It was resynced on 2026-07-02 and is labelled "BENCH.md @ throttlekit 1.6.1, 2026-06-30".
- `src/plugins/rewrite-links.mjs` rewrites the design docs' repo-relative links back to `github.com/AmeyaBorkar/throttlekit/blob/main/…` so cross-repo links resolve.

The site markets a **fourth published package too**: `src/changelog.ts` documents `throttlekit-server` (the gRPC service door + Lens, npm), so the "3-repo family" framing is true for repos but the **site advertises 3 registry packages** (`throttlekit`, `throttlekit-server`, `throttlekit-py`). As of 2026-10-12 their `current` fields (1.7.0 / 0.4.4 / 0.5.1) match the npm and PyPI latest versions.

## Architecture & how it's structured

Standard Astro layout. Annotated tree (assets collapsed):

```
throttlekit-site/
├─ astro.config.mjs        site origin (throttlekit.in), sitemap(), markdown:
│                          {syntaxHighlight:false, rehypePlugins:[rewriteLinks]}
├─ package.json            3 deps; scripts: dev/build/preview/sync:design
├─ tsconfig.json           extends astro/tsconfigs/strict
├─ scripts/
│  └─ sync-design.mjs      copy docs/design/*.md from the monorepo (local only)
├─ public/                 ~5 MB of static assets
│  ├─ frames/ f_0001..f_0120.webp   120 hero scrub frames (2.5 MB)
│  ├─ feather.mp4          hero source clip (2.4 MB; not used at runtime — frames are)
│  ├─ og.jpg, favicon.svg, apple-touch-icon.png, robots.txt
└─ src/
   ├─ config.ts            SITE constants (name, tagline, links, license), NAV, FOOTER
   ├─ content.config.ts    content collections: `design` (glob NN-*.md) + `blog`
   ├─ blog.ts              getPosts(), postNeighbors(), date formatters
   ├─ changelog.ts         CHANGELOG[] — curated release highlights for 3 packages (now through 1.7.0 / 0.4.4 / 0.5.1)
   ├─ design.ts            DESIGN_GROUPS / DESIGN_DOCS — sidebar manifest + neighbors
   ├─ data/bench.ts        BENCH_META + benchmark tables (mirror of BENCH.md, resynced 07-02)
   ├─ plugins/
   │  └─ rewrite-links.mjs rehype plugin: rewrite design-doc anchors
   ├─ layouts/
   │  ├─ Base.astro        HTML shell: <head> meta/OG, pre-paint theme script, fonts, Nav, Footer, site-wide JS
   │  ├─ DesignLayout.astro 3-column docs shell (sidebar / main / optional TOC)
   │  └─ PostLayout.astro  blog-post chrome (header, .prose body, prev/next)
   ├─ components/
   │  ├─ Nav.astro         top bar (hero/solid variants) + mobile menu JS + theme toggle
   │  ├─ Footer.astro      5-column footer from config.FOOTER
   │  └─ Background.astro   opt-in ambient canvas (roaming glow + cursor-repel embers; hidden in light theme)
   ├─ styles/
   │  └─ global.css        the entire design system (590 lines, incl. the light-theme block)
   ├─ content/
   │  ├─ design/  01-..18-*.md   18 component design docs (synced from core repo, then edited site-side)
   │  └─ blog/    6 *.md         editorial posts (frontmatter-fronted)
   └─ pages/                routes (see inventory below)
```

**Content collections** (`src/content.config.ts`):
- `design` — `glob` loader on `[0-9][0-9]-*.md`, schema `{ title?: string }` (bodies are plain markdown; nav/labels live in `design.ts`).
- `blog` — `glob` on `*.md`, schema `{ title, date (coerced), description, tags[], author=default, draft=false }`.

## Page/route inventory

13 page routes + 1 RSS endpoint. Two are dynamic (`getStaticPaths`). No routes were added or removed since June.

| Route | File | Purpose |
|---|---|---|
| `/` | `pages/index.astro` (319 ln, largest) | Home. "Act I" = inline canvas scroll-scrub hero (120 frames, 5 text "beats"); "Act II" = capability comparison table (now also listing Lens and Policy Plans rows), the two-engine cards, BENCH numbers, "what's new" band (pulls latest post + latest release), 30-sec quickstart code. |
| `/tale` | `pages/tale.astro` | TALE engine — the LLM **cost axis** (token-budget escrow). Problem framing, reserve→meter→reconcile mechanism, three layers, Python `debit` sample, proof + honest edges. |
| `/gale` | `pages/gale.astro` | GALE engine — provable distributed **leasing / overshoot bound**. Window-coupling keystone, three modes, the two-tier perf lever, EOQ lease sizing, the trilemma, weighted-fair escrow, TLA⁺ proof references. |
| `/lens` | `pages/lens.astro` | ThrottleKit Lens — terminal dashboard for **binding-axis attribution**. Eight tabbed views, the Monitor gRPC door + Prometheus + gRPC health, honest edges. |
| `/benchmarks` | `pages/benchmarks.astro` | Reproducible benchmark tables from `data/bench.ts` (in-process, Redis, Postgres head-to-heads, and now the 1.6.1 perf-sweep before/after table), methodology, machine spec, reproduce commands. |
| `/compare` | `pages/compare.astro` | Capability matrix vs express-rate-limit / rate-limiter-flexible / @upstash/ratelimit, "where an incumbent ties or wins," "pick which when." |
| `/status` | `pages/status.astro` | "State of ThrottleKit": current versions of all 3 packages, shipped-capability board, stable-vs-experimental frontier, how it's verified. |
| `/changelog` | `pages/changelog.astro` | Per-package curated release highlights from `changelog.ts`, with a tiny inline-`code` renderer. |
| `/about` | `pages/about.astro` | Why it exists, the four build commitments, the two engines, status, MIT/who. |
| `/design` | `pages/design/index.astro` | Design-docs landing: grouped cards from `DESIGN_GROUPS`. |
| `/design/[slug]` | `pages/design/[slug].astro` | One rendered design doc; `getStaticPaths` from the `design` collection; sidebar + on-page TOC (h2/h3) + prev/next + "View source ↗" back to the core repo. |
| `/blog` | `pages/blog/index.astro` | Blog index (published posts, newest first; first item is a "feature"). |
| `/blog/[slug]` | `pages/blog/[slug].astro` | One post; `getStaticPaths` builds **all** posts incl. drafts (draft previewable by direct URL but excluded from index/RSS/neighbors). |
| `/404` | `pages/404.astro` | "Over the limit" not-found page with scoped styles. |
| `/rss.xml` | `pages/rss.xml.js` | RSS feed endpoint via `@astrojs/rss`, fed by `getPosts()`. |

Plus `sitemap-index.xml` generated by `@astrojs/sitemap` (referenced from `public/robots.txt`).

**Nav vs. route note:** `src/config.ts` `NAV` (unchanged) exposes TALE, GALE, Lens, Design, Benchmarks, Blog, Docs↗ (wiki), GitHub↗. `/compare`, `/about`, `/status`, `/changelog` are **reachable only via the footer / cross-links**, not the top nav (a deliberate "keep the spine lean" choice per the config comment). Docs (`SITE.wiki`) is an **external** link to the core repo's GitHub wiki, not an on-site route.

## Notable components & interactive bits (demos/playgrounds)

- **The scroll-scrub hero** (`index.astro`, inline IIFE): the signature interactive piece. It `fetch`es 120 WebP frames, decodes via `createImageBitmap`, shows a loading bar, then draws frames onto a full-viewport `<canvas>` driven by scroll progress over a `660vh` track. Five text "beats" cross-fade in opacity windows mapped to scroll progress. On mobile/touch/reduced-motion it bails to a stacked-text reveal (no video, no canvas). Manual `history.scrollRestoration` reset. On desktop in light mode, the loader, hero navbar and `#track` stay pinned to the dark palette so the beats remain white over the black footage. This is the closest thing to a "playground," but **there is no live rate-limit playground** — no interactive demo where you tweak limits and watch admissions. The "demos" are static rendered ASCII mock-ups (e.g. the Lens "DENIALS BY AXIS" bar chart in `lens.astro`, the bound pseudo-code callouts) and copy-paste code blocks.
- **Theme toggle (new).** An inline script in `<head>` (`Base.astro`) resolves the theme before first paint, so there is no flash of the wrong theme. It uses the saved `localStorage['tk-theme']` value if present, otherwise `prefers-color-scheme`, and sets `html[data-theme="light"]` plus the `theme-color` meta (`#f6f3ec`). `Nav.astro` adds a sun/moon button that flips the attribute and saves the choice. Because of the `prefers-color-scheme` fallback, a visitor whose OS is set to light now gets the light theme by default. The earlier commit's note that the "live site is unchanged until the theme is set" therefore applies only to visitors with dark OS settings.
- **`Background.astro`**: an opt-in (`bg` prop) ambient canvas — a breathing radial gold glow that randomly roams the screen, plus 24 cursor-repelling ember particles tracking mouse motion-energy. Desktop-only, disabled under `(pointer:coarse)` and reduced-motion via both CSS and a JS guard, and now hidden under the light theme. "Finalized via /bg-demo" per the comment, but **no `/bg-demo` page exists in the repo** (a dev-only tuning page that wasn't committed).
- **Site-wide JS in `Base.astro`**: copy-to-clipboard for any `[data-copy]` pill and `.code-copy` button (used by all the `$ npm i…` / `pip install…` chips), and an `IntersectionObserver` scroll-reveal for `.section > .wrap` (with reduced-motion / no-IO fallbacks).
- **`Nav.astro` JS**: mobile hamburger toggle; active-section detection by pathname prefix; the theme toggle (which also sits beside the hamburger on mobile).
- **`DesignLayout.astro`**: a 3-column docs shell that conditionally renders a right-hand TOC only when the page passes a `toc` slot; sidebar groups come from `DESIGN_GROUPS`.

## Content & messaging (cross-reference to the core)

The site is a faithful, **detailed** marketing surface for the core library's claims. Key messages, all traceable to the library's own docs/tests it cites:

- **Two engines no other limiter has:** **TALE** (Temporally-Accounted Learned Escrow) governs the **cost axis** — a streaming LLM token-budget escrow with overshoot bounded by debit granularity (Δ = 0 at per-token metering, independent of `max_tokens`/concurrency); **GALE** (Globally-Accounted Learned Escrow) provides **provable distributed leasing** where window-coupling collapses worst-case global admissions to exactly `Limit` for any fleet size, machine-checked in TLA⁺ (`spec/GaleWindowCoupledLeasing.tla`, `MaxAdmitted == Limit`).
- **The unifying tagline:** "Meter what your LLM spends. Prove what your fleet admits." / "Admitted ≤ Limit." The home hero now adds "Every other limiter just counts requests. ThrottleKit governs *rate, concurrency, and cost*, each behind a bound you can prove."
- **Benchmark headline numbers.** `data/bench.ts` was resynced on 07-02 and now matches the core's current `BENCH.md`:
  - a full GCRA `checkSync` in **170 ns** (5.9M ops/s, ~1 B/op), was 169 ns;
  - **71.2k ops/s** via two-tier leasing over Redis at batch 100, **~97×** the strict 735 ops/s path (was 66.4k, ~85×, 783).

  A stale head-to-head claim was corrected at the same time. On Redis, `/compare` used to say ThrottleKit "has a slightly tighter tail" (778 vs 752 ops/sec). It now says the two are within noise on both throughput (745 vs 754) and tail, with "no consistent edge either way." The Postgres lever is now "~27×" over rate-limiter-flexible's 342 ops/sec upsert, down from "~35×". The site stays unusually **honest** about caveats: it repeatedly flags that Redis/Postgres absolute latency reflects Docker-on-Windows loopback and won't transfer, and that on a single Postgres counter `rate-limiter-flexible`'s UPSERT *wins*.
- **18 design docs** mirror the core's component reference (core model, algorithms, 6 stores, two-tier/leasing, cost axis, concurrency, unified admission, federation, adapters, observability, overload/security, config/CLI, wire protocol, gRPC server, **Python client**, policy plans, replay, Lens). Since July they also document the 1.7.0 hardening invariants: decision-finiteness, the config-parser nesting cap, and the cost/bucket/sketch overflow guards.
- **6 blog posts:**
  - "Introducing ThrottleKit Lens"
  - "Policy Plans — a terraform plan for your limits"
  - "Scaling to a fleet without changing a line of client code"
  - "Every release, from the start" (extended through 1.7.0)
  - **new:** "A hardening sweep — throttlekit 1.6.0 and server 0.4.x" (2026-06-29)
  - **new:** "A performance sweep, then a hardening pass: 1.6.1 to 1.7.0" (2026-07-01)

  The second new post's figures (mutation score 65.4% → 77.5%, fuzz findings F1–F5, 1,551 storeless tests) match the core's `CHANGELOG.md`, `JOURNEY.md` and `research/hardening/MUTATION-BACKLOG.md`.
- **Changelog/status** track three registry packages with per-release highlights: `throttlekit` (npm, **v1.7.0**), `throttlekit-server` (npm, **v0.4.4**), `throttlekit-py` (PyPI, **v0.5.1**). All three are current against the registries.

Useful cross-check for the core docs: the cost-axis Python sample uses `ServiceBackend("localhost:50051").debit(policy, key, tokens=…)` and the site claims it's "verified against the throttlekit-py API" — a concrete signature to validate against that repo (it matches `ServiceBackend.debit(policy, key, tokens=1)`).

## Styling & design system

- **Single hand-written stylesheet**, `src/styles/global.css` (590 lines, up from 493), imported once in `Base.astro`. **No Tailwind, no CSS framework, no preprocessor.** A handful of components/pages add scoped `<style>` blocks (`Background.astro`, `404.astro`).
- **Design tokens** via CSS custom properties: near-black `--bg:#000`, off-white `--fg:#f3f1ec`, muted/faint greys, a single accent `--glow:#efe1bd` ("a whisper of pale gold"), and three Google Fonts — **Instrument Serif** (display), **Inter** (sans), **JetBrains Mono** (code/eyebrows), loaded via `<link>` with preconnect.
- **Light theme (new).** `html[data-theme="light"]` flips the six core tokens to a "paper" palette: `--bg:#f6f3ec`, `--fg:#1b1915`, and a deep amber `--glow:#9a7616` "legible on paper". Targeted overrides follow for surfaces, code blocks, tints and gold washes that were authored as literal colors. The commit states that the dark rules are byte-identical to before.
- **Aesthetic:** cinematic monochrome. The hero nav uses `mix-blend-mode:difference` to invert over the canvas; content pages use a sticky blurred-backdrop nav. Reusable building blocks: `.section/.wrap`, `.kicker/.lead`, `.tile/.trio/.duo/.quad`, `.callout/.bound`, `.cmp` comparison table, `.dtable` benchmark tables, `.prose` (shared by design docs **and** blog), `.band/.card` stat cards, `.cmd` copy chips, `.topbar-right`/`.theme-toggle`.
- **Accessibility/perf-conscious:** reduced-motion handled in both CSS and JS for the hero, background, and reveals; the theme toggle has an `aria-label`; `compressHTML`, `inlineStylesheets:'auto'`, and markdown `syntaxHighlight:false` (code blocks styled by `.prose` instead of a Shiki theme) keep output lean.

## Tech stack & dependencies

- **Astro 6.4.2** — static site generator. Config (`astro.config.mjs`): `site: 'https://throttlekit.in'`, `integrations:[sitemap()]`, `build.inlineStylesheets:'auto'`, `devToolbar:{enabled:false}`, `compressHTML:true`, and `markdown:{ syntaxHighlight:false, rehypePlugins:[rewriteLinks] }`.
- **`@astrojs/sitemap` 3.7.3** — the only integration.
- **`@astrojs/rss` 4.0.18** — used in the `/rss.xml` route.
- **TypeScript** — strict (`extends astro/tsconfigs/strict`); used for the config/data modules (`config.ts`, `blog.ts`, `changelog.ts`, `design.ts`, `data/bench.ts`, `content.config.ts`).
- **Node ESM tooling** — `rewrite-links.mjs` (rehype) and `sync-design.mjs` (fs copy) are plain `.mjs`.
- **Zero runtime dependencies, zero UI framework, zero CSS toolchain.** The entire `dependencies` block is the three Astro packages above; there are no `devDependencies` declared in `package.json`. The dependency set is unchanged since June.

## Build / dev / deploy

Scripts (`package.json`):
- `npm run dev` → `astro dev --port 4321 --host`
- `npm run build` → `astro build` (emits static `dist/`)
- `npm run preview` → `astro preview --port 4321`
- `npm run sync:design` → `node scripts/sync-design.mjs` (local convenience to refresh design docs from the monorepo; now destructive to the site-side edits, see above)

**Hosting:** README says "Static build (`dist/`), hosted on Vercel. No server runtime." That matches the build output, but **there is no Vercel config or adapter committed** — no `vercel.json`, no `@astrojs/vercel`, no `.vercel/` (it's git-ignored). Deployment is therefore by Vercel's zero-config static detection (or external), not anything in-repo. `.gitignore` excludes `dist/`, `.astro/`, `.vercel/`, `node_modules/`. `.gitattributes` normalizes text to LF and marks `.webp/.mp4/.png/.jpg/.ico/.woff2` binary. `public/robots.txt` welcomes all crawlers (incl. AI bots — GPTBot, ClaudeBot, PerplexityBot, etc.) and points the sitemap at `https://throttlekit.in/sitemap-index.xml`. `Base.astro` carries a `google-site-verification` meta tag.

## Status, completeness & notable gaps

**Complete and polished as a marketing/docs site, and current.** All 13 routes render real content, and the content pipeline (collections, RSS, sitemap, link-rewriting, design sync) is fully wired. As of the 07-02 sync, the versions, changelog and benchmark numbers match the published packages and the core's `BENCH.md`; nothing has shipped upstream since. Notable observations and gaps:

- **No LICENSE file** despite MIT branding throughout; links point at the core repo's LICENSE.
- **No live/interactive rate-limit playground.** Despite a code-heavy, demo-flavored presentation, every "demo" is a static rendered mock-up or copy-paste snippet. The interactive features are the cosmetic scroll-scrub hero and, now, the theme toggle.
- **Dangling `/bg-demo` reference.** `Background.astro` says it was "finalized via /bg-demo," but that page isn't in the repo (a dev-only tuning page left uncommitted). Not user-visible.
- **Manual cross-repo sync is still a staleness risk, now with a twist.** The June doc's example (bench numbers pinned to throttlekit 1.0.0) is fixed: `bench.ts` now mirrors `BENCH.md`. Two problems remain:
  - The design docs are no longer verbatim copies (all 18 differ from the core), so `sync:design` would undo the em-dash pass and the site-only edits. Keeping them current now means hand-merging rather than copying.
  - `BENCH_META` labels the numbers "BENCH.md @ throttlekit 1.6.1, 2026-06-30". Only the perf-sweep rows date from then. The in-process, Redis and Postgres tables come from `BENCH.md`'s 2026-06-10 / 1.4.0 run.
- **The 2.4 MB `feather.mp4` ships in `public/` but isn't referenced by any page** (the hero uses the 120 extracted WebP frames). It's dead weight in the deployed bundle unless intentionally kept as the frame source.
- **Heavy first-load on `/`** — 120 WebP frames (2.5 MB) are fetched on the home page; mitigated by the mobile bail-out and a loader, but it is the dominant payload.
- **Quiet since 2026-07-02.** This matches the core, which has shipped nothing since 1.7.0.

## README vs. code

The README has not changed since June.

| README claim | Code reality |
|---|---|
| "Built with Astro: static output" | ✅ True — Astro 6.4.2, default static build. |
| "~zero client JS apart from one hydrated canvas island that scroll-scrubs the feather hero" | ⚠️ Mostly true, loosely worded, and looser than in June. It's **inline vanilla JS**, not an Astro hydrated *island* (no framework, no `client:*`). It also isn't the *only* JS. `Base.astro` (clipboard, reveal observer, and now a pre-paint theme resolver), `Nav.astro` (mobile menu plus the theme toggle) and `Background.astro` (ambient canvas) all ship inline scripts too. |
| "the rate limiter that meters what your LLM spends and proves what your fleet admits" | ✅ Matches `SITE.tagline`/`SITE.description` and the home hero. |
| `public/frames/` = "120 WebP frames extracted from it (scroll-scrub)" | ✅ Exactly 120 `f_0001..f_0120.webp`; `feather.mp4` is the "hero source clip." Note: the mp4 ships but is unused at runtime. |
| "Static build (`dist/`), hosted on Vercel. No server runtime." | ⚠️ Output is static `dist/` with no server runtime (true), but **no Vercel config/adapter is committed** — "hosted on Vercel" is an operational fact, not anything in the repo. |
| README links ThrottleKit to `github.com/AmeyaBorkar/throttlekit` | ✅ Consistent with `SITE.gh`. But the **canonical site origin in `astro.config.mjs` is `https://throttlekit.in`**, not a GitHub Pages/Vercel URL — worth knowing the live domain. |
| README "Structure" lists only `public/` and `src/{pages,styles}` | ⚠️ Understated — omits the substantial `src/{content,layouts,components,data,plugins}` and `scripts/`, which are where most of the actual structure lives. |
| (not in README) light/dark theme | ℹ️ The theme toggle and light palette, the most visible feature change since June, aren't mentioned in the README. |

---

*Doc generated 2026-06-16, refreshed 2026-10-12 by reading the source at C:\Users\ameya\Documents\GreenfeildProject\research\website\site. Source of truth: https://github.com/AmeyaBorkar/throttlekit-site*
