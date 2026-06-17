# markdown-viewer

> A single Python package (`markdown_viewer`) that exposes one rendering core through three surfaces — a FastAPI REST API, a tkinter desktop app, and a browser web client — built around a hand-written, pure-stdlib, line-based markdown engine (no external markdown library) and four hand-tuned design-token themes. The same core drives all three surfaces and self-contained HTML exports.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/markdown-viewer |
| Visibility | Public |
| Category | web |
| Primary language(s) | Python (engine + all 3 surfaces); JS/CSS/HTML for the web client |
| Local path | `C:\Users\ameya\Documents\Helper\MarkdownViewer` |
| Default branch | `main` |
| Lines of code (computed) | 3,632 lines of Python across 22 `.py` files; plus web client: 450 JS + 450 CSS + 118 HTML = 1,018 non-Python lines |
| Source files (computed) | 22 Python files (incl. 3 tests), 1 JS, 1 CSS, 1 HTML template, `sample.md` |
| PyPI package | Packaged as `markdown-viewer` v`0.1.0` (in `pyproject.toml`); **not verified as published to PyPI** — install is documented from git only |
| Key dependencies | Core/desktop: **none** (pure stdlib). API: `fastapi>=0.110`, `uvicorn[standard]>=0.27`. Dev: `pytest>=7.4`. Web client (CDN, runtime): KaTeX 0.16.10, Mermaid 10.9.1, Google Fonts |
| License | MIT (`Copyright (c) 2026 Ameya Borkar`) |
| Last commit | `a9c7694` — *docs: write polished README covering all three surfaces and the CLI* (2026-05-08); 16 commits total, all on 2026-05-07/08 |

---

## What it actually is

`markdown_viewer` is a Python package with a clean separation between a **dependency-free core** and three **surfaces** that consume it:

- **`core/`** — the markdown engine (`renderer.py`), a theme registry (`themes.py`), a document analyzer (`analyzer.py`: frontmatter + stats + TOC), and a self-contained HTML exporter (`exporter.py`). Zero third-party imports.
- **`api/`** — a FastAPI app exposing render / analyze / themes / sandboxed file ops / export over HTTP, and also serving the web client.
- **`desktop/`** — a tkinter three-pane editor (outline · editor · live preview) with a separate, simpler markdown→`Text`-widget renderer.
- **`web/`** — a static HTML/CSS/JS client served by the API; it renders by round-tripping through `/api/render` so it always matches the other surfaces.

A unified CLI (`__main__.py`) ties it together with subcommands `desktop`, `api`, `web`, `render`, and `export`. Running `python -m markdown_viewer` with no subcommand launches the desktop app.

The README's framing ("three surfaces from one package, same engine") is broadly accurate, with one important nuance: the **desktop preview does NOT use the core `renderer.py`** — it has its own independent block/inline parser in `desktop/preview.py` that renders into a tkinter `Text` widget. So strictly there are **two** markdown implementations in the repo (the core HTML renderer used by API + web + export, and the tkinter preview renderer). They share only the compiled block-detector regexes imported from `renderer.py`.

## Architecture & how it's structured (one engine, three surfaces; annotated tree)

```
MarkdownViewer/
├── pyproject.toml              # packaging: name="markdown-viewer" v0.1.0, MIT, py>=3.10
│                               # deps=[] ; optional [api]=fastapi+uvicorn, [dev]=pytest
│                               # console script: markdown-viewer = markdown_viewer.__main__:main
├── requirements.txt            # fastapi + uvicorn only (API surface); core is dep-free
├── LICENSE                     # MIT, 2026 Ameya Borkar
├── README.md                   # polished marketing-style readme
├── sample.md                   # showcase document
├── markdown_viewer/
│   ├── __init__.py             # __version__ = "0.1.0"
│   ├── __main__.py             # CLI: desktop|api|web|render|export (199 LOC)
│   ├── core/                   # ── THE ENGINE (zero deps) ──
│   │   ├── __init__.py         # docstring only (no re-exports — see README-vs-code)
│   │   ├── renderer.py         # hand-written markdown → HTML engine (580 LOC)
│   │   ├── themes.py           # 4 themes as design-token dataclasses (217 LOC)
│   │   ├── analyzer.py         # frontmatter parser + stats + TOC tree (278 LOC)
│   │   └── exporter.py         # self-contained HTML page builder (316 LOC)
│   ├── api/                    # ── SURFACE 1: REST API (FastAPI) ──
│   │   ├── main.py             # create_app(): CORS, static mount, "/" serves web (58 LOC)
│   │   ├── routes.py           # all endpoints across 5 routers (236 LOC)
│   │   ├── models.py           # pydantic request/response models (90 LOC)
│   │   └── workspace.py        # sandboxed-directory path resolver (61 LOC)
│   ├── desktop/                # ── SURFACE 2: tkinter desktop ──
│   │   ├── app.py              # window shell, menus, commands, Find dialog (540 LOC)
│   │   ├── editor.py           # Text editor + line-number gutter (213 LOC)
│   │   ├── preview.py          # SECOND markdown parser → styled Text widget (409 LOC)
│   │   └── theme.py            # maps Theme tokens onto ttk "clam" styles (148 LOC)
│   └── web/                    # ── SURFACE 3: website (served by api) ──
│       ├── templates/index.html  # app shell: topbar, panes, palette, toasts (118 LOC)
│       └── static/
│           ├── css/premium.css   # styling using --mv-* tokens (450 LOC)
│           └── js/app.js          # client logic; renders via /api/render (450 LOC)
└── tests/                      # stdlib unittest (44 test methods total)
    ├── test_renderer.py        # 26 tests
    ├── test_analyzer.py        # 11 tests
    └── test_themes_exporter.py #  7 tests
```

Dependency direction: `api`, `desktop`, and `web`(via `api`) all depend on `core`; `core` depends on nothing but the stdlib (`html`, `re`, `dataclasses`, `math`).

## The markdown engine — exactly which constructs it parses and how (the algorithm)

File: `markdown_viewer/core/renderer.py`. It is a **line-based, single-pass block scanner** (not a CommonMark-compliant AST parser). `Renderer.render(source)` normalises line endings, splits on `\n`, and walks lines with an index `i`, dispatching each line to a block handler in a fixed priority order, then runs a separate **two-phase inline pass** on the text content.

**Block-level constructs (checked in this order per line):**

1. **Fenced code blocks** — `_RE_FENCE` matches ```` ``` ```` or `~~~` (3+), optional language tag; collects until a matching closer. A `mermaid` language emits `<div class="mermaid">…</div>`; otherwise `<pre data-lang="…"><code class="language-…">…</code></pre>`. Body is HTML-escaped.
2. **ATX headings** — `# … ######` (1–6), trailing `#`s stripped; emits `<h1..h6>` with an auto-slugged `id` (via `slugify`, deduped with `-2`, `-3`… suffixes). Records `(level, text, slug)` for the TOC.
3. **Setext headings** — a text line followed by `===` (→ h1) or `---` (→ h2).
4. **Horizontal rules** — `---`, `***`, `___` (3+).
5. **Display math** — `$$` on its own line, collected until the closing `$$`, emitted as `<div class="math math-display">` (escaped) for client-side KaTeX. Gated by `enable_math`.
6. **Blockquotes** — `> …` with **lazy continuation** (non-blank glued lines join the quote); the quote body is **recursively rendered** through `render()` and wrapped in `<blockquote>`.
7. **Lists** — unordered (`-`, `*`, `+`) and ordered (`\d+.` / `\d+)`). Ordered lists honour a non-1 `start=` attribute. **GFM task lists** (`[ ]` / `[x]`) become `<li class="task-list-item">` with a disabled checkbox; the `<ul>`/`<ol>` gets `class="task-list"` if any item is a task. Supports single-level continuation via indentation but **does not produce nested `<ul>`/`<ol>`** — nested indentation is folded into the item text (see gaps).
8. **GFM tables** — triggered when a line contains `|` and the next line matches `_RE_TABLE_SEP`. Parses an alignment row (`:--`, `--:`, `:--:` → left/right/center via inline `style="text-align: …"`), a `<thead>`, and `<tbody>` rows.
9. **Indented code blocks** — lines starting with 4 spaces → `<pre><code>` (escaped, no language).
10. **Paragraphs** — the fallback; consecutive non-blank lines until a blank line or a block starter.

**Inline pass (`_inline`) — a two-phase stash/restore design:**

Phase 1 "stashes" anything whose payload must be opaque to later passes, replacing each match with a `\x00<index>\x00` placeholder, in this order: **code spans** → **inline math** (`$…$`) → **images** (`![alt](src "title")`) → **links** (`[label](href "title")`, with the label recursively inline-processed) → **angle-bracket autolinks** (`<https://…>`, `<mailto:…>`) → **bare URLs** (`https://…`, gated by `enable_autolinks`). Phase 2 HTML-escapes the remaining literal text (when `safe_mode`), then applies **strong** (`**`/`__`), **emphasis** (`*`/`_`), **strikethrough** (`~~`, gated), and **hard breaks** (2+ trailing spaces → `<br />`). Finally placeholders are restored. This ordering is what makes `[**bold**](url)` and "don't double-wrap an already-linked URL" work.

**Safety:** `safe_mode=True` (default) means raw HTML is escaped, not passed through — verified by tests (`<script>` → `&lt;script&gt;`). With `safe_mode=False`, raw HTML is emitted verbatim.

**Options** (`RenderOptions` dataclass, frozen): `enable_tables`, `enable_strikethrough`, `enable_task_lists`, `enable_math`, `enable_mermaid`, `enable_autolinks`, `heading_anchors`, `safe_mode`, `hard_wrap`.

**Result** (`RenderResult`): `html`, `headings: list[(level, text, slug)]`, `word_count` (regex word tokens). Helpers: `slugify()` and module-level `render()` one-shot.

**Verdict:** A genuine, reasonably complete GFM-subset engine, but line-oriented and regex-driven rather than a spec-conformant parser. It deliberately favours clarity (per its own docstring) over CommonMark edge-case fidelity.

## Surface 1: REST API (FastAPI) — endpoints table

App factory: `api/main.py` → `create_app()` adds permissive CORS (`allow_origins=["*"]` by default, `allow_credentials=True`), mounts `/static`, serves the web client at `/`, and calls `register_routes()`. Routes live in `api/routes.py` across five `APIRouter`s. Title "Markdown Viewer API", version pulled from `__version__`.

| Method | Path | Purpose | Notes |
|---|---|---|---|
| GET | `/health` | Heartbeat | `{status:"ok", version}` |
| POST | `/api/render` | Markdown → HTML | Returns `html`, `headings[]`, `word_count`; accepts all render flags |
| POST | `/api/analyze` | Document analysis | Returns `frontmatter`, `stats`, `toc` tree |
| GET | `/api/themes` | List themes | Each with a 5-colour `palette` subset |
| GET | `/api/themes/{theme_id}` | One theme | 404 on unknown id |
| GET | `/api/themes.css` | Full theme CSS | `text/css`, hidden from schema; consumed by the web client |
| GET | `/api/files/list?path=` | Sandboxed dir listing | Hides dotfiles; dirs first; 400 on escape, 404 if missing |
| GET | `/api/files/read?path=` | Sandboxed file read | UTF-8 only (415 otherwise) |
| POST | `/api/files/save` | Sandboxed file write | Optional `create_dirs`; 400 if parent missing |
| POST | `/api/export/html` | Self-contained HTML | `PlainTextResponse` as `text/html` |
| POST | `/api/export/html/download` | Same + attachment | `Content-Disposition: attachment` (URL-quoted filename) |
| GET | `/` | Web client | Serves `templates/index.html` (not in schema) |

**Sandbox (`api/workspace.py`):** a `Workspace(root)` resolves client paths under a single directory; `candidate.relative_to(root)` raises `PermissionError` (→ HTTP 400) on `..`/absolute escapes. Root defaults to CWD, overridable via `--workspace` CLI flag or `MARKDOWN_VIEWER_WORKSPACE` env var. OpenAPI docs at `/docs` and `/redoc` (FastAPI defaults).

This matches the README's endpoint table accurately.

## Surface 2: Desktop app (tkinter)

Entry: `python -m markdown_viewer` (default) or `… desktop` → `desktop/app.py:run()` → `MarkdownViewerApp`.

- **Layout:** menu bar (File/Edit/View/Help) + nested `ttk.Panedwindow`s giving a three-pane workspace: **Outline sidebar · Editor · Preview**, plus a status bar (word count, reading time, cursor position `Ln/Col`, file label, current theme). Window 1360×860, min 720×460.
- **Editor (`editor.py`):** a `tk.Text` with undo, a painted **line-number gutter** (`_LineGutter` canvas synced to scroll/cursor), Tab/Shift-Tab block indent (4 spaces), and change/cursor callbacks. Preview is debounced ~120 ms via `after`.
- **Preview (`preview.py`):** a **separate, simpler markdown renderer** that walks blocks and inserts styled text into a read-only `tk.Text` using tags (`h1`–`h6`, `code_block`, `quote`, `list`, `task_done/open`, `table_header/cell`, etc.). It reuses the core's compiled block regexes (`_RE_FENCE`, `_RE_ATX_HEADING`, …) but has its own inline tokenizer. Notable rendering choices: tables drawn with `│`/`─` glyphs; images shown as `🖼 alt`; task items as `☑`/`☐`. Does **not** render math/HTML — it's a typographic approximation, not the HTML engine.
- **Outline:** clickable headings (from `analyze().toc`) that search the editor text and scroll to the heading.
- **Find (`FindDialog`):** modal next/previous search with highlight.
- **Commands:** New, Open, Save, Save As, Export HTML (uses the core `export_html`), theme switch (re-themes the whole window via `theme.py`), toggle outline, about.
- **Shortcuts (bound in code):** `Ctrl+N/O/S`, `Ctrl+Shift+S`, `Ctrl+E`, `Ctrl+Q`, `Ctrl+F`, `Ctrl+\`. (README also lists `Ctrl+Z/Y` for undo/redo — those exist as menu accelerators but are not explicitly `bind()`-ed; tk's `undo=True` handles them natively.)
- **Theming (`theme.py`):** forces the ttk `clam` theme and overrides Frame/Label/Button/Combobox/Entry/Paned/Scrollbar element options from the shared `Theme` tokens, plus `option_add` global colours. `PLATFORM_FONTS` favours Windows fonts (Segoe UI / Cambria / Cascadia Mono) with graceful fallback.

## Surface 3: Website

Served at `/` by the API; assets under `web/`.

- **`templates/index.html`:** the app shell — topbar (New / Open / Save / Export, theme `<select>`, outline & zen toggles), workspace (outline sidebar · editor `<textarea>` · splitter · preview `<article class="mv-content">`), status bar, a hidden **command palette**, and a toast tray. Pulls `/api/themes.css`, `/static/css/premium.css`, Google Fonts, and KaTeX from CDN.
- **`static/js/app.js` (450 LOC, no framework, no build step):** the renderer **never runs in the browser** — `renderNow()` debounces (180 ms) and POSTs the source to `/api/render` and `/api/analyze` in parallel, injects the returned HTML, updates stats/TOC, then runs KaTeX `renderMathInElement`. Features: theme switch persisted to `localStorage`; draft autosave (`mv:draft:v1`); Save (downloads `.md` Blob); Export (POST `/api/export/html`, downloads result); Open (FileReader); draggable splitter (persisted); zen mode; outline navigation; **command palette** (`Ctrl+K`) with filterable actions; `beforeunload` guard. Hotkeys: `Ctrl+K/S/O/E/N`, `Ctrl+\`, `F11`.
- **`static/css/premium.css` (450 LOC):** consumes the `--mv-*` custom properties from `/api/themes.css`; CSS-grid layout, topbar/panes/palette/toast styling.

Caveat: the web "Save" downloads a file via the browser; it does **not** call the API's `/api/files/save` sandbox routes (those exist but appear to be used by no shipped client — they're API capabilities, not wired into the web UI).

## Themes — the four themes and how theming works

Defined in `core/themes.py` as a frozen `Theme` dataclass and a `THEMES` dict, in preference order:

| id | Label | Mode | bg | accent |
|---|---|---|---|---|
| `ivory` | Ivory | light (default) | `#FAF8F3` (warm off-white) | `#9C6A2F` |
| `graphite` | Graphite | dark | `#16171A` | `#D6A75A` |
| `sepia` | Sepia | light | `#F4ECD8` (paper) | `#A1531B` |
| `midnight` | Midnight | dark | `#0B0D10` (deep blue-black) | `#7DB7E0` |

Each theme is a bundle of **design tokens**: `bg`, `surface`, `surface_alt`, `text`, `text_muted`, `text_subtle`, `accent`, `accent_soft`, `border`, `code_bg`, `code_text`, `link`, `selection`, `shadow`, plus shared serif/sans/mono font stacks (Source Serif 4 / Inter / JetBrains Mono with platform fallbacks).

**One token set, three consumers:**
- **Web + HTML export** → `Theme.to_css()` emits `--mv-*` CSS custom properties; `themes_css()` binds the default to `:root` and each theme to `html[data-theme="<id>"]`, so switching is a single attribute toggle. The API serves this at `/api/themes.css`.
- **Desktop** → `Theme.tk` exposes the colour subset; `desktop/theme.py` maps them onto ttk styles.

`get_theme()` falls back to `ivory` for unknown/None ids (tested). This matches the README's "four hand-tuned themes, tokens shared across all three surfaces" claim exactly.

## Tech stack & dependencies (verify the no-external-markdown-lib claim)

- **Language/runtime:** Python ≥3.10 (uses `X | Y` unions, `list[...]` generics). Web client is vanilla ES modules + CSS, no bundler.
- **Core + desktop:** **zero third-party dependencies.** Confirmed by `pyproject.toml` (`dependencies = []`) and by grepping the package — the only imports in `core/`/`desktop/` are stdlib (`html`, `re`, `dataclasses`, `math`, `tkinter`, `pathlib`, `os`).
- **API only:** `fastapi>=0.110`, `uvicorn[standard]>=0.27`, plus `pydantic` (transitively via FastAPI; imported directly in `models.py`).
- **Dev:** `pytest>=7.4` (though the tests themselves use stdlib `unittest`).
- **Web client runtime (CDN, not Python deps):** KaTeX 0.16.10 (math), Mermaid 10.9.1 (diagrams, via the exporter), Google Fonts.

**"No external markdown library" — VERIFIED.** There is no `markdown`, `mistune`, `markdown2`, `commonmark`, or `marko` anywhere in dependencies or imports. The only grep hits for `import … markdown … render` are inside **sample-document text** (the welcome doc in `desktop/app.py` and `web/app.js` shows `from markdown_viewer.core import render` as example content). Math/Mermaid rendering is delegated to client-side JS (KaTeX/Mermaid) — the Python engine only emits passthrough markup, so the claim holds for markdown parsing itself.

## Build / run / test (actual commands; packaging)

Packaging: setuptools via `pyproject.toml`; package auto-discovery includes `markdown_viewer*`, excludes `tests*`; web assets bundled as package-data. Console entry point `markdown-viewer = markdown_viewer.__main__:main`.

```bash
# Install (editable). Core+desktop need nothing; [api] pulls FastAPI/uvicorn.
python -m pip install -e ".[api]"

# Desktop app (also the default with no subcommand)
python -m markdown_viewer
python -m markdown_viewer desktop

# REST API + bundled web client at http://127.0.0.1:8765/
python -m markdown_viewer api
python -m markdown_viewer web            # same, but opens a browser tab
# flags: --host --port --reload --workspace --open

# Render a file to an HTML fragment on stdout (or -o FILE)
python -m markdown_viewer render sample.md

# Build a self-contained HTML page
python -m markdown_viewer export sample.md -o out.html -t graphite
# export flags: -t/--theme {ivory,graphite,sepia,midnight} --title --no-toc --no-math --no-mermaid

# Tests
python -m unittest discover -s tests
```

## Tests

Stdlib `unittest` (not pytest, despite the `[dev]` extra). **Exactly 44 test methods** across 3 files (computed) — the README's "44 unit tests" claim is **accurate**:

- **`test_renderer.py` (26):** ATX/Setext headings, anchor slugs + dedup, bold/italic/code, links/autolinks/bare-URLs (incl. no-double-wrap), strikethrough, HTML-escape-by-default and `safe_mode=False` passthrough, images, fenced + indented code, ordered/unordered/task lists, table alignment, blockquote lazy continuation, horizontal rules, inline + block math, Mermaid passthrough, `slugify`, word count.
- **`test_analyzer.py` (11):** frontmatter (typed scalars, inline lists, block lists, quoted values, unterminated-block left alone), stats (word/paragraph counts, reading time at 220 wpm), TOC tree shape + `render_toc_html`.
- **`test_themes_exporter.py` (7):** all four themes present, `get_theme` fallback, `themes_css` emits `:root` + `[data-theme]` blocks, exporter title/theme-attr/body, embedded inline styles, TOC inclusion/exclusion.

Coverage is on the **core only** — there are no tests for the API routes, the workspace sandbox, the desktop app, the second (tkinter) preview renderer, or the web client.

## Status, completeness & notable gaps

- **Working & solid:** the core HTML engine (broad GFM subset, well-tested), themes, analyzer, exporter, and the FastAPI surface look complete and coherent. CLI covers all surfaces.
- **Maturity:** classified "Development Status :: 4 - Beta"; all 16 commits landed within ~20 minutes on 2026-05-08 (a single authored push), so this is a freshly-scaffolded project, not one with iterative history.
- **Engine limitations (by design):** no true nested lists (`<ul>` inside `<li>`) — nested indentation is folded into the parent item's text; no reference-style links/footnotes; no HTML block handling beyond escape/passthrough; not CommonMark-conformant (regex/line-based).
- **Two renderers, one repo:** the desktop preview (`preview.py`) is an independent approximation of the core renderer; they can drift. The README's "same engine renders all three" is true for API + web + export but **not** literally for the desktop preview.
- **Unused capability:** the sandboxed `/api/files/*` routes are not consumed by the bundled web client (which uses browser download/upload). They're a real API feature awaiting a client.
- **CORS:** ships wide-open (`allow_origins=["*"]` + `allow_credentials=True`) — fine for a local single-user tool, not for public deployment.
- **PyPI:** declared as package `markdown-viewer` v0.1.0 but README only documents a `git clone` install; publication to PyPI is not evidenced in the repo.

## README vs. code

- **Accurate:** three surfaces / one package; pure-stdlib engine with no external markdown lib; the four named themes; the full REST endpoint table; the "44 unit tests" count; the listed CLI commands and flags; desktop and web keyboard shortcuts; KaTeX/Mermaid passthrough.
- **Slight overstatement — "the same engine drives all three":** API, web, and exports do share the core `renderer.py`, but the **desktop preview uses a different, simpler renderer** (`desktop/preview.py`). "What you see in the browser is exactly what you'll get from the desktop preview" is therefore approximate, not literal.
- **Sample-doc import is wrong:** both the desktop and web welcome documents show `from markdown_viewer.core import render`, but `core/__init__.py` is a bare docstring and does **not** re-export `render` — that snippet would raise `ImportError`. The real path is `from markdown_viewer.core.renderer import render`. (This is sample copy, not shipped code, but it's a documentation bug.)
- **Theme count vs. prose:** `themes.py`'s own module docstring twice says "**Three** ranges are bundled by default" then lists four — a stale comment; the code defines four themes (matching the README).
- **Undo/redo shortcuts:** README lists Ctrl+Z/Ctrl+Y for the desktop; these appear as menu accelerators and rely on Tk's built-in `undo=True`, but are not separately `bind()`-ed like the other shortcuts.
- **README repo URLs:** README/`pyproject` use lowercase `github.com/ameyaborkar/...`; the canonical repo is `github.com/AmeyaBorkar/markdown-viewer` (GitHub is case-insensitive, so both resolve).

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\Helper\MarkdownViewer. Source of truth: https://github.com/AmeyaBorkar/markdown-viewer*
