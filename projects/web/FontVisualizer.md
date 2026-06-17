# FontVisualizer

> A single-file, zero-dependency, dark-themed font playground. You type a sample string and see it rendered live across ~132 curated Google Fonts (grouped sans / serif / display / handwriting / mono), drag your favourites into a persistent "collection" tray, then open a full-screen "Studio Board" to isolate the shortlist and fine-tune color (HSV wheel), size, letter/line spacing, indentation, alignment, weight/italic, and seven CSS-text-shadow "lighting" presets. Fonts are lazy-loaded via the Google Fonts CSS API as cards scroll into view; everything (collection, background, board settings) persists in `localStorage`.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/FontVisualizer |
| **Visibility** | Private |
| **Category** | web |
| **Primary language(s)** | HTML (single file containing embedded CSS + vanilla JS) |
| **Local path** | `C:\Users\ameya\Documents\Helper\FontVisualizer` |
| **Default branch** | `main` |
| **Lines of code (computed)** | 1,253 total lines in `index.html` (51,497 bytes). Breakdown: HTML markup ~lines 1–8 + 397–531, CSS in `<style>` lines 9–395 (~386 lines), JS in `<script>` lines 533–1251 (~718 lines) |
| **Source files (computed)** | 1 — `index.html` (the only non-`.git` file in the repo) |
| **Key dependencies** | None bundled. Runtime: Google Fonts CSS2 API (`fonts.googleapis.com` / `fonts.gstatic.com`). Browser APIs: `IntersectionObserver`, Canvas 2D, Pointer Events, Drag-and-Drop, `localStorage`, Clipboard |
| **License** | None — no `LICENSE` file present |
| **Last commit** | `21cff72` — "Initial commit: Font Visualizer" (2026-06-14) — the only commit in the repo |

## What it actually is

FontVisualizer is a **client-side typography browser and comparison tool**, delivered as a single self-contained `index.html` (~51 KB) with no build step, no package manager, and no external JS/CSS files. Everything — markup, styling, and logic — lives in that one file.

It does three things:

1. **Specimen grid** — renders a responsive grid of cards, one per font in a hard-coded catalog of curated Google Fonts. Each card shows your sample text in that typeface plus the font name and category. Filterable by category chips, searchable by name, with a live size slider and a preview-background switcher.
2. **Collection tray** — a right-hand "My Collection" panel you build by dragging cards in (or clicking `+`, or pressing the `+` key over a hovered card). The collection is an ordered list of font names persisted to `localStorage`, with Copy / Clear actions.
3. **Studio Board** — a full-screen overlay that "isolates" either your whole collection or a single font, stacking them as lines on a styleable stage. It includes a hand-rolled HSV color wheel (drawn on `<canvas>`), separate text/background color targets with an invert button, sliders for size / letter-spacing / line-height / indent, alignment, bold/italic toggles, font-name label toggle, and seven `text-shadow`-based "lighting" presets (soft shadow, neon glow, spotlight, emboss, long shadow, backlit) with intensity/angle controls.

It does **not** load custom/uploaded font files (no `FontFace` constructor, no drag-drop of `.ttf`/`.woff`), does **not** inspect glyph tables, and does **not** expose variable-font axes — it only requests `wght@400;700` from Google. It is a preview/comparison/specimen tool, not a font-internals inspector.

## Architecture & how it's structured (annotated tree)

```
FontVisualizer/
└── index.html          # 1,253 lines — the entire app
    ├── <head>          # lines 1–396
    │   ├── meta + <title>Font Visualizer</title>
    │   ├── <link rel="preconnect"> to fonts.googleapis.com / fonts.gstatic.com   (lines 7–8)
    │   └── <style>     # lines 9–395 — all CSS
    │       ├── :root design tokens (dark palette + --surface/--ink theming vars)  (10–27)
    │       ├── .app CSS grid layout (header / main / tray) + 880px responsive break (38–62)
    │       ├── header, .sample-input, .controls, .swatches, .chips, .toggle switch (64–168)
    │       ├── .grid + .card font-specimen styling (170–214)
    │       ├── .tray / .dropzone / .tray-item collection panel (216–279)
    │       └── .board-overlay Studio Board: head/side/stage, wheel, sliders, presets (281–394)
    ├── <body>          # lines 397–1253
    │   ├── .app
    │   │   ├── header.top  — title, #sampleInput, #search, #size, BG #swatches+#bgColor,
    │   │   │                 #onlyCollected toggle, #count, #chips category bar   (400–431)
    │   │   ├── main.main   — empty #grid container (filled by JS)                  (433–435)
    │   │   └── aside.tray  — #dropzone, #emptyState, Board/Copy/Clear buttons      (437–454)
    │   ├── #board .board-overlay  — Studio Board markup (head / side controls / stage) (459–531)
    │   │   ├── color group: tabs (Text/Background/⇄), #wheel canvas, #valSlider,
    │   │   │                #hexInput, #nativeColor, #boardSwatches
    │   │   ├── layout group: #bSize #bLetter #bLine #bIndent, #alignSeg, bold/italic, labels
    │   │   └── lighting group: #lightPresets (7 buttons), #lightIntensity, #lightAngle
    │   └── <script>    # lines 533–1251 — all JS (see walkthrough)
    └── (no README, no LICENSE, no package.json, no .gitignore, no node_modules)
```

There is exactly **one source file**. No tooling, no transpilation, no framework.

## Code walkthrough — the HTML/CSS/JS, what each part does

### HTML / CSS
- **Layout** is a two-column CSS grid (`.app`, line 39): `1fr` main area + a fixed `340px` tray, with a sticky header row; collapses to a single column below `880px` (line 53) where the tray becomes a sticky bottom panel.
- **Theming** uses CSS custom properties (`:root`, line 10). A second set — `--surface`, `--ink`, `--ink-muted` (lines 23–26) — is mutated at runtime by the background switcher so card surfaces and text-ink color flip for light vs. dark preview backgrounds.
- The **Studio Board** (`.board-overlay`, line 285) is `display:none` until JS adds `.open`, then becomes a fixed full-screen grid with a 330px control sidebar and a flexible stage.

### JavaScript (lines 533–1251)
- **Font catalog (lines 537–583).** `FONTS` is a hard-coded array of `[name, category]` tuples `.map`ped to `{name, category}` objects. The catalog holds **132 fonts**: 42 sans, 27 serif, 28 display, 20 handwriting, 15 mono (computed by category). `CATEGORIES` and `CAT_LABEL` drive the chip bar.
- **State (lines 588–601).** A single `state` object holds `text`, `size`, `search`, `category`, `onlyCollected`, `collection` (loaded from `localStorage` key `fontviz.collection.v1`), and `bg` (key `fontviz.bg.v1`). `BG_PRESETS` is 7 preset background colors.
- **`luminance(hex)` (line 604)** computes WCAG relative luminance; **`applyBackground()` (line 614)** sets `--surface` and auto-chooses light/dark `--ink` based on whether luminance < 0.5, then syncs the swatch UI and persists.
- **`loadCollection` / `saveCollection` (629–635)** serialize the collection array to `localStorage`.
- **Lazy font loading (lines 642–669).** `ensureFontLoaded(name)` guards a `Set`, then injects a `<link rel="stylesheet">` to `https://fonts.googleapis.com/css2?family=<name>:wght@400;700&display=swap`; an `onerror` handler retries without the weight axis for single-weight families. An `IntersectionObserver` (200px rootMargin) calls this when a card scrolls near the viewport and then un-observes it.
- **Grid rendering (671–748).** `visibleFonts()` filters by category + search + collected-only. `buildCard(font)` creates a draggable card: a `.sample` `<p>` with `font-family:"<name>", sans-serif`, the name/category meta, an Add (`+`/`✓`) button calling `toggleCollect`, and an Isolate (`⛶`) button calling `openBoard([font])`. Cards track hover for the keyboard shortcut and register `dragstart`/`dragend`. `renderGrid()` rebuilds the grid into a `DocumentFragment` and updates the count.
- **Tray rendering (750–797).** `renderTray()` re-renders the collection list (each item shows the sample + name + remove `✕`), ensures each collected font is loaded, and shows/hides the empty state.
- **Collection logic (799–831).** `toggleCollect`, `addToCollection`, and `syncCardState` (which updates a single card's collected styling without rebuilding the grid, using `cssEscape` to safely build the `[data-font="…"]` selector).
- **Drag & drop (833–853).** Tray dropzone handles `dragenter/over/leave/drop`, reads the dropped font name from `dataTransfer`, and adds it.
- **Controls wiring (855–937).** Live handlers for the sample-text input (mutates text nodes in place — no re-render), size slider (mutates `fontSize` in place), search, collected-only toggle, dynamically built category chips, dynamically built BG swatches + custom color picker, Clear (with `confirm()`), Copy (newline-joined names to clipboard with success feedback), and a global `keydown` so `+`/`=` adds the hovered card.
- **Studio Board (939–1244).** Separate `boardState` (key `fontviz.board.v1`; only style settings persist — fonts/text/target are excluded on save). Includes:
  - **Color math** — `hsvToRgb`, `rgbToHsv`, `hexToRgb`, `rgbToHex`, `normHex`, `isHex` (979–1017).
  - **HSV wheel** — `drawWheel()` (1020) renders an angular hue / radial saturation wheel pixel-by-pixel into the canvas `ImageData` at full value, with a 1.2px anti-aliased edge. `positionMarker`, `updateValueTrack`, and `wheelPointer` (with Pointer Events + pointer capture for drag) map clicks/drags back to HSV. A value slider controls brightness; a hex input and native OS color input round-trip the color.
  - **Targets** — Text / Background tabs plus an invert button swap which color the wheel edits.
  - **Layout/lighting binding (1107–1126)** — `bindRange` wires the numeric sliders; alignment segment and lighting presets toggle active state.
  - **Stage rendering (1143–1215)** — `renderBoardLines()` stacks one `.board-line` per font; `applyBoardStyles()` writes all the inline styles; `computeStageBg()` adds a radial-gradient spotlight; `computeShadow()` builds the `text-shadow` string per lighting preset (soft/glow/emboss/dramatic long-shadow loop/backlit) using intensity + angle.
  - **`openBoard` / `closeBoard` (1227–1244)** — show/hide the overlay, sync controls, lock body scroll (`noscroll`). `Escape` closes it.
- **Init (1246–1250).** `drawWheel()`, `applyBackground(state.bg, false)`, `renderGrid()`, `renderTray()` run on load. There is no `DOMContentLoaded` wrapper — the `<script>` is at the end of `<body>`, so the DOM already exists.

## How it works — the font-loading / rendering mechanism

- **No `FontFace` API, no Web Font Loader, no canvas text rendering.** Text is rendered with ordinary DOM `<p>` elements whose `style.fontFamily` is set to the font name (with a `sans-serif` fallback). The browser does all glyph rendering.
- **Font delivery is the Google Fonts CSS2 API.** For each font, a `<link rel="stylesheet">` pointing at `fonts.googleapis.com/css2?family=<Name>:wght@400;700&display=swap` is injected into `<head>`. Google returns `@font-face` rules whose `src` references `fonts.gstatic.com` woff2 files; the browser fetches and applies them. `&display=swap` means the fallback shows immediately and swaps when the real font arrives.
- **Lazy loading via `IntersectionObserver`** (rootMargin 200px) means font CSS is only requested when a card approaches the viewport — avoiding ~132 up-front requests. Collected/tray and board fonts are force-loaded via `ensureFontLoaded`.
- **Single-weight fallback:** if the `:wght@400;700` request errors, a retry without the weight axis is injected (line 651).
- **The HSV color wheel is the only `<canvas>` usage** — it is purely a color-picker UI control, not used for type rendering.
- **Persistence** is three `localStorage` keys: `fontviz.collection.v1` (collection array), `fontviz.bg.v1` (preview background), `fontviz.board.v1` (board style settings only).

## Features — what the user can actually do

- Type any sample string and preview it live across all ~132 fonts (default sample: "The quick brown fox jumps").
- Filter by category chips (All / Sans-serif / Serif / Display / Handwriting / Monospace).
- Search fonts by name (live).
- Adjust preview font size with a 16–80px slider.
- Switch the preview background among 7 presets or pick any custom color; text ink auto-flips light/dark for legibility (WCAG luminance).
- Build a collection by dragging cards into the tray, clicking `+`, or pressing `+`/`=` over a hovered card; remove via `✕`; "Collected only" toggle.
- Copy the collection's font names (newline-separated) to the clipboard; Clear all (with confirm).
- Open the **Studio Board** for the whole collection, the first 6 visible fonts (if collection empty), or a single isolated font (the `⛶` per-card button).
- On the board: pick text/background colors via HSV wheel + brightness slider + hex/native pickers + 16 swatches, invert text↔bg, set size (12–220px), letter-spacing, line-height, indent, alignment, bold, italic, toggle font-name labels, and apply one of 7 lighting presets with intensity + angle.
- Everything (collection, background, board settings) is restored on reload.

## Tech stack

- **HTML5** single page; **CSS3** (custom properties, CSS Grid, `backdrop-filter`, gradients) embedded in `<style>`.
- **Vanilla JavaScript (ES6+)** — no framework, no libraries, no bundler. Uses arrow functions, destructuring, template literals, `Set`, `Object.assign`.
- **Browser APIs:** `IntersectionObserver`, Canvas 2D (`createImageData`/`putImageData`), Pointer Events (+ pointer capture), HTML Drag-and-Drop, `localStorage`, `navigator.clipboard`.
- **External service:** Google Fonts CSS2 API (the only network dependency).

## Build / run (actual)

There is **no build step** — no `package.json`, no bundler config, no transpiler. To run:

1. Open `index.html` directly in any modern browser (double-click or `file://`), **or**
2. Serve it from any static server, e.g. `python -m http.server` then visit the page.

An internet connection is required for fonts to render in their true typefaces (Google Fonts is fetched at runtime); offline, everything still works but fonts fall back to `sans-serif`. The Clipboard API may require a secure context (`https://` or `localhost`) in some browsers, so serving locally is slightly more reliable than `file://`.

## Status, completeness & notable gaps

**Status:** Functional and self-contained at first commit; the feature set described above is fully implemented in code (not stubbed).

Notable observations / gaps (from reading the code):
- **No custom font upload.** Despite the "visualizer" name, you cannot load your own `.ttf`/`.woff`; the catalog is a fixed list of Google Fonts. No `FontFace` constructor is used.
- **No variable-font axes or weight selection** beyond requesting 400/700 from Google; the grid/tray render at default weight, and only the board exposes a binary bold toggle.
- **Catalog count nuance:** the second-field category extraction sums to 132 fonts (42+27+28+20+15); a naive line/bracket grep reported 131–133 due to entries wrapping across source lines. The authoritative figure is **132**.
- **Single shared `boardState` for align**: the board persists style settings but intentionally drops `fonts`/`text`/`target` on save (lines 955, 959) so the board always opens fresh against the current collection — by design.
- **`cssEscape` only escapes `"`** (line 831); font names with other CSS-selector-special characters could in theory break the `[data-font]` selector, but none in the current catalog contain such characters.
- **Accessibility:** the board sets `aria-hidden` correctly, but the app relies heavily on hover + drag interactions; the `+` keyboard shortcut is the main keyboard affordance.
- No tests, no CI, no error reporting beyond the font-load `onerror` retry.

## README vs. code

There is **no README** in the repository (confirmed: the only non-`.git` file is `index.html`; no `README*`, `LICENSE*`, or `package.json` exist). Therefore there are no documentation claims to reconcile against the code — the single `<title>` and the in-page subtitle ("Type text, preview it in every font, drag your favorites into the tray.") are the only self-description, and both accurately match the implemented behavior.

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\Helper\FontVisualizer. Source of truth: https://github.com/AmeyaBorkar/FontVisualizer*
