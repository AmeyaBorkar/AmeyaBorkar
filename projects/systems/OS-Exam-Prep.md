# OS-Exam-Prep

> A dependency-free, single-page web app for revising an Operating Systems university final (VIT-style). It bundles six units of hand-written Markdown — cheat sheets, ~62 model exam answers (with fully worked numericals), and 20 "mind maps" — into one giant `content.js` blob that a vanilla HTML/CSS/JS shell renders client-side. Two small Python scripts compile the Markdown into the bundle; mind maps are pre-rendered to static boxed HTML by Python (not by any JS mind-map library).

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/os-exam-prep |
| **Visibility** | Private |
| **Category** | systems |
| **Primary language(s)** | Markdown (content, ~12.9k LOC tracked) · Python 3 (build, 294 LOC) · HTML/CSS/vanilla JS (SPA shell, 1,032 LOC) |
| **Local path** | `C:\Users\ameya\Documents\VITassignments&study\OSstudyMaterial\website` |
| **Default branch** | `main` |
| **Lines of code (computed)** | ~14.2k tracked source: 12,922 LOC across 36 content `.md` files + 294 LOC Python + 1,032 LOC `index.html`. Generated `content.js` is 895 KB on one line (gitignored). |
| **Source files (computed)** | 40 git-tracked files: 36 `.md` (4 README/site + 32 content), 2 `.py`, 1 `index.html`, 2 `.bat`, `.gitignore`. Plus untracked generated artifacts (`content.js`, 20 `*.html` mind maps, `preview-unit3-algos.html`). |
| **Key components** | `index.html` (SPA shell + all CSS + router) · `build_site.py` (bundler) · `build_mindmaps.py` (Markdown→boxed-HTML renderer) · `content/unit{1..6}/` (source Markdown) · `content.js` (generated bundle) |
| **License** | None (SPDX). README states: "Personal study material. Not for redistribution as course content." |
| **Last commit** | `d1753f7` — "docs: add getting-started guide for collaborators" (2026-05-13) |

## What it actually is

A **static, build-once single-page application**. There is no framework, no `package.json`, no Node, no server requirement. The architecture is:

1. **Source content** lives as plain Markdown under `content/unit1..6/` — three kinds: `cheatsheet.md`, `answers.md`, and a `mindmaps/*.md` folder.
2. **A Python build step** (`build_site.py`, run manually) reads every Markdown file, renders the mind-map Markdown into static HTML via `build_mindmaps.py`, JSON-serializes everything, and writes one file: `content.js`, which is literally `window.CONTENT = {...big JSON...};`.
3. **`index.html`** is a ~1,000-line self-contained shell: all CSS is inlined in one `<style>` block, all JS in one `<script>` block. It `<script src="content.js">`-loads the bundle, reads `window.CONTENT`, and renders pages on demand using hash-based routing (`#mindmaps/unit3/...`).

So at runtime it is pure client-side: open `index.html` (even via `file://`), the browser parses the 895 KB `content.js`, and the JS router paints the requested unit/section. The only network dependencies are three CDN scripts: `marked` (Markdown → HTML for cheat sheets/answers) and `highlight.js` (code-block syntax highlighting, dark+light themes). Mind maps need no JS library because they are pre-baked to HTML at build time.

Despite home-page copy promising "interactive, zoomable" mind maps with "zoom (scroll), pan (drag), and click nodes to expand/collapse," the actual mind-map renderer produces **static** colored cards with nested bullet lists — no zoom/pan/collapse JS exists. The only interactivity on a mind-map page is a "Density" button that cycles font size between compact/normal/large CSS classes. (See README vs. code.)

## Architecture & how it's structured

```
website/                              (= repo root, github.com/AmeyaBorkar/os-exam-prep)
├── index.html              1,032 LOC  SPA shell: inlined CSS (dark/light themes, mind-map
│                                      card styles, markdown styles) + vanilla-JS router,
│                                      theme toggle, collapsible sidebar, page renderers
├── content.js              895 KB     GENERATED (gitignored): window.CONTENT = {readme,
│                            1 line     cheatsheet{}, answers{}, mindmaps{}} — all content baked in
├── build_site.py             122 LOC  Bundler: reads content/**.md + ../Markdowns/
│                                      README_MasterIndex.md → renders mindmaps → writes content.js
├── build_mindmaps.py         172 LOC  Markdown→HTML: parses # / ## / ### + nested bullets into
│                                      a root box + color-rotated branch cards (also has a
│                                      standalone main() that writes *.html next to each .md)
├── content/                            SOURCE Markdown (this is what's edited)
│   ├── unit1/  (Introduction to OS)
│   │   ├── cheatsheet.md      700 LOC  Sections A/B/C/E: wall sheet, triggers, mnemonics, rapid revision
│   │   ├── answers.md         838 LOC  Section D: 10 model answers (uses ### Q1. headings)
│   │   └── mindmaps/                   3 maps: 01-overview, 02-types-of-os, 03-services-and-syscalls
│   ├── unit2/  (Processes & Threads)  cheatsheet 920 · answers 1,143 · 4 mindmaps
│   ├── unit3/  (CPU Scheduling)       cheatsheet 708 · answers 780 · 3 mindmaps
│   ├── unit4/  (Deadlocks)            cheatsheet 599 · answers 904 · 3 mindmaps
│   ├── unit5/  (Memory Management)    cheatsheet 635 · answers 779 · 4 mindmaps
│   └── unit6/  (I/O & File Mgmt)      cheatsheet 688 · answers 830 · 3 mindmaps
│       └── mindmaps/*.md              (also *.html siblings, gitignored — regenerated each build)
├── open.bat                            Launcher: `start index.html` (opens via file://)
├── serve.bat                           Launcher: `python -m http.server 8000` + opens browser
├── README.md                 153 LOC  Project + clone/run guide + getting-started for collaborators
└── .gitignore                          Ignores __pycache__, content/unit*/mindmaps/*.html,
                                        preview-*.html, editor/OS junk  (NOTE: does NOT ignore content.js)
```

Content totals (computed): **20 mind-map Markdown files** (3+4+3+3+4+3 across units 1–6) and **62 model-answer questions** (10/10/11/10/11/10). Each unit has exactly one cheat sheet.

Note on git tracking: 40 files are tracked. The generated artifacts `content.js` (895 KB) and the 20 `content/unit*/mindmaps/*.html` files and `preview-unit3-algos.html` are **untracked** — the `.html` mind maps and `preview-*` are explicitly `.gitignore`d, and `content.js` is currently untracked in the working tree as well (the `.gitignore` does NOT list it, so it is simply not committed). This means a fresh clone has no `content.js` until someone runs `python build_site.py`.

## Code walkthrough (the important parts)

### `index.html` — the SPA shell (1,032 LOC)
- **CSS (lines ~15–631):** one inlined stylesheet. Defines a GitHub-flavored dark theme as default via CSS custom properties under `:root,[data-theme="dark"]`, with a full `[data-theme="light"]` override block. Styles cover: header, collapsible sidebar, mind-map browser grid (`.mindmap-grid`/`.mindmap-card`), the static mind-map renderer (`.mm-root`, `.mm-grid`, `.mm-card`, a 10-color palette `.mm-card-blue` … `.mm-card-lime`), rendered-Markdown body (`.md-body` with tables, blockquotes, code), home hero, loading spinner, error box, custom scrollbars, and `@media (max-width:800px)` mobile rules.
- **Markup (lines ~633–712):** header (brand + theme toggle), a `<nav class="sidebar">` with hardcoded sections — Mind Maps, Cheat Sheets, Model Answers, Extras (Master Index) — each listing the six units with `data-route` attributes, and a `<main>` with breadcrumb toolbar + `#content` pane.
- **JS router (lines ~717–1030):**
  - `handleRouteChange()` reads `location.hash` (default `#home`) and dispatches in `renderRoute()` by splitting on `/`: `home`, `readme`, `mindmaps/<unit>[/<slug>]`, `cheatsheet/<unit>`, `answers/<unit>`.
  - `renderMarkdown(md)` strips YAML frontmatter, calls `marked.parse`, then schedules a `setTimeout` pass calling `hljs.highlightElement` on `.md-body pre code`.
  - `renderMindmapList(unit)` builds clickable cards from `window.CONTENT.mindmaps[unit]` (title/desc/slug). `renderMindmapView(unit,slug)` injects the pre-rendered `map.html` into a `.mindmap-host` and exposes a `cycleDensity()` button.
  - `renderMdView(kind,unit)` renders cheat sheets/answers via `renderMarkdown`.
  - Theme is persisted in `localStorage["os-theme"]` (default dark) and swaps the two highlight.js stylesheets by toggling their `disabled` flags. Sidebar collapse persists in `localStorage["os-sidebar-collapsed"]` with a `Ctrl+B`/`Cmd+B` shortcut. Sidebar section headers toggle a `collapsed` class.
  - Every "not loaded" branch shows an error telling the user to "Run `build_site.py` first" — confirming the build dependency.

### `build_mindmaps.py` — Markdown → boxed HTML (172 LOC)
- `md_inline()` does a tiny inline-Markdown pass (`**bold**`, `*italic*`, `` `code` ``, `-->`/`->` → `→`) with HTML escaping.
- `parse_mindmap()` walks lines into a tree: `# ` → root, `## ` → a branch (card), `### ` → a bold "heading" leaf, and `-`/`*` bullets become nested nodes via an indent stack. It strips `---\n…\n---\n` frontmatter first.
- `render_branch()`/`render_node()` emit `<section class="mm-card mm-card-{color}">` cards with nested `<ul class="mm-sublist">`; `render_mindmap()` wraps them: a `.mm-root` box, a `.mm-stem`, then a `.mm-grid` of cards, color-rotated through a 10-name `PALETTE`.
- Has its own `main()` that writes a `.html` next to each `.md` — but `build_site.py` imports `render_mindmap` and renders fresh in-memory each bundle, so those on-disk `.html` files are vestigial scratch output (and gitignored).

### `build_site.py` — the bundler (122 LOC)
- Reads `content/unit{1..6}/cheatsheet.md` and `answers.md` verbatim, collects mind maps via `collect_mindmaps_for_unit()` (each gets `{slug, title, desc, html}` where `html` is the freshly rendered boxed HTML), and reads `../Markdowns/README_MasterIndex.md` as the `readme` ("Master Index") entry.
- `parse_mindmap()` (a lighter one than the renderer's) extracts a title (first `# `) and a one-line `desc` for the card list.
- Writes `window.CONTENT = ` + minified JSON + `;` to `content.js`, then prints stats (cheat sheets, model answers, mind-map count, README OK, output KB).
- **Dependency outside the repo:** `README_MasterIndex.md` lives at `../Markdowns/` — i.e. *outside* the website folder/repo. It exists locally (328 LOC, 19.5 KB) but is not part of this repo, so a clone of just `os-exam-prep` will build with an empty Master Index.

### Content files (the bulk of the value)
- **Cheat sheets** are structured "Sections A/B/C/E": A = visual wall sheet (lots of ASCII-art boxes in fenced code blocks), B = trigger points, C = mnemonics, E = last-30-minute rapid revision.
- **Answers** are "Section D" exam-style model answers in 15-mark format with definition lines, bodies, ASCII diagrams, comparison tables, and "key points to mention in exam." Heading style is inconsistent: unit 1 uses `### Q1.`; units 2–6 use `## Q1.` / `## Q1)`.
- **Mind-map Markdown** files all carry a `markmap:` YAML frontmatter block (`colorFreezeLevel`, `initialExpandLevel`, `maxWidth`) — a leftover from being authored for the markmap tool — but the site ignores that and uses the Python boxed renderer.

## Content coverage — OS topics (from the actual content)

Six units, mapped from `UNIT_TITLES` in `index.html` and confirmed by the Markdown:

- **Unit 1 — Introduction to OS:** what an OS is (resource allocator / control program / interface), computer-system architecture, OS services (the 9 core services), system calls, types of OS. Mind maps: overview, types-of-os, services-and-syscalls. No numericals.
- **Unit 2 — Process & Thread Management:** process concept (program vs process, memory layout), 2/5/7-state process models, PCB structure + process table + context switch, process creation/termination, threads & multithreading, concurrency & mutex. (Largest unit: 920-LOC cheat sheet, 1,143-LOC answers, 4 mind maps.) No numericals.
- **Unit 3 — CPU Scheduling:** scheduler types (long/short/medium-term), optimization criteria (CPU util, throughput, TAT, WT, RT, fairness), preemptive vs non-preemptive, and the algorithms FCFS / SJF / SRTF / Round Robin / Priority with advantages/disadvantages and Gantt-chart numericals. Mind maps include an "algorithms deep dive."
- **Unit 4 — Deadlocks:** definition, Coffman conditions, prevention, and handling strategies, including **Banker's algorithm** safety-check worked tables. Mind maps: overview, coffman-and-prevention, handling-strategies.
- **Unit 5 — Memory Management:** paging & segmentation, address translation (4 KB / 1 KB page sizes), page replacement (FIFO / LRU), virtual memory & thrashing. (Most questions: 11.) 4 mind maps.
- **Unit 6 — I/O & File Management:** I/O & buffering, disk scheduling (FCFS / SSTF / SCAN / C-SCAN head-movement numericals), file systems & directories. 3 mind maps.

Numericals with step-by-step worked arithmetic appear in units 3, 4, 5, 6 (units 1–2 are conceptual only), matching the home-page "6-Unit Map" table.

## Tech stack

- **No framework / no build toolchain for JS** — vanilla HTML, one inlined `<style>`, one inlined `<script>`.
- **Python 3** (standard library only: `json`, `re`, `html`, `pathlib`, `sys`) for the two build scripts.
- **CDN runtime libs:** `marked` (Markdown rendering) and `highlight.js@11.9.0` (code highlighting, github-dark + github light themes) — loaded from jsDelivr.
- **Routing:** hash-based, hand-rolled (`location.hash` + `hashchange`).
- **State:** `localStorage` for theme + sidebar-collapse preferences.
- **Content format:** Markdown (GitHub-flavored), compiled to a single `window.CONTENT` JSON blob.

## Build / run / deploy

**Build (required before first run):**
```bash
python build_site.py     # reads content/**.md, renders mind maps, writes content.js
```
`build_mindmaps.py` can be run standalone (`python build_mindmaps.py`) to dump `*.html` next to each mind-map `.md`, but this is optional — `build_site.py` renders in-memory.

**Run (any of):**
```bash
# 1) Open directly — works because content.js loads via <script src>, valid under file://
double-click index.html        # Windows
open index.html                # macOS
./open.bat                     # bundled file:// launcher

# 2) Local HTTP server
./serve.bat                    # Windows: python -m http.server 8000 + opens browser
python -m http.server 8000     # then visit http://localhost:8000
```

**Deploy:** No deploy config present — no `vercel.json`, no GitHub Actions/Pages workflow, no Dockerfile. It is a static folder; any static host (or `file://`) works once `content.js` is built. README documents clone-and-run for collaborators (gh auth, clone, serve) but not hosted deployment.

## Status, completeness & notable gaps

- **Functionally complete as a study tool:** all six units have a cheat sheet, an answers file, and mind maps; 62 model answers and 20 mind maps are present and substantive (12.9k LOC of content). The SPA renders all routes.
- **Build artifact not committed:** `content.js` is not tracked and is *not* gitignored either — a clone has no `content.js`, so the site shows "Run build_site.py first" errors until built. This is the single biggest gap for a new collaborator.
- **External build dependency:** `build_site.py` reads `../Markdowns/README_MasterIndex.md`, which lives outside the repo. The "Master Index" / Extras page will be empty for anyone who clones only `os-exam-prep`.
- **Vestigial code paths:** `build_mindmaps.py.main()` writes `.html` files that the live site never reads (they're regenerated and inlined by `build_site.py`); these are gitignored.
- **Heading-format inconsistency** in answers (unit 1 `###`, units 2–6 `##`) — cosmetic, renders fine, but breaks the README's `### Q1.` search tip for 5 of 6 units.
- **No tests, no linting, no CI.**
- **Overstated interactivity:** mind maps are static; the "zoom/pan/collapse" promised in the home-page copy and README is not implemented (only a font-density toggle exists).

## README vs. code

| Claim in README / UI copy | Reality in code |
|---|---|
| "**20** mind maps" | Correct — 20 `.md` files (3+4+3+3+4+3). Home-page hero says "20+ interactive mind maps." |
| Model answers "**60+**" / home says "60+" | 62 actual (`## Q…` / `### Q…` headings: 10/10/11/10/11/10). Accurate. |
| Mind maps are "**interactive, zoomable** … zoom (scroll), pan (drag), click nodes to expand/collapse" (home page) | **False.** Renderer (`build_mindmaps.py`) emits static boxed HTML; the only interactivity is a CSS font-density toggle. README's own Features section more honestly says "no JS rendering library, no zooming weirdness, just bounded boxes." |
| "search `### Q1.` etc. inside any answers page" | Only true for **unit 1**. Units 2–6 use `## Q1.`/`## Q1)` (H2). |
| `content.js` "(~870 KB)" / "~870 KB content bundle" | Actual on-disk size is **895 KB** (~874 KB measured by build); close, slightly understated. |
| "~22 KB SPA shell" | `index.html` is ~34 KB on disk (1,032 lines incl. inlined CSS+JS). Understated. |
| Project-layout box: `content.js` "generated bundle … all markdown + mindmap HTML" | Correct, but the README does not flag that `content.js` is **uncommitted**, so a fresh clone is broken until `build_site.py` runs. |
| "Works offline after the first load — CDN libs cache automatically" | Plausible but unverified; the first load needs network for `marked` + `highlight.js`. No offline/service-worker code exists. |
| Licence: "Personal study material. Not for redistribution" | No `LICENSE` file / SPDX identifier in the repo; it's a prose note only. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\VITassignments&study\OSstudyMaterial\website. Source of truth: https://github.com/AmeyaBorkar/os-exam-prep*
