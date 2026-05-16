# Markdown Viewer

> A premium, minimalist Markdown viewer that ships three surfaces from one Python package — a FastAPI REST API, a tkinter desktop app with live preview, and a website with command palette and export. Pure-stdlib markdown engine (no third-party markdown library), four hand-tuned themes, self-contained HTML exports.

**Repository:** [`AmeyaBorkar/markdown-viewer`](https://github.com/AmeyaBorkar/markdown-viewer)  
**Category:** Developer Tool / Documentation  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** MIT License  
**Created:** 2026-05-07  
**Last pushed:** 2026-05-07  
**Metadata updated:** 2026-05-13  
**Size (GitHub reported):** 56 KB  

---

## What it is (one-paragraph version)

A premium, minimalist Markdown viewer that ships three surfaces from one Python package — a FastAPI REST API, a tkinter desktop app with live preview, and a website with command palette and export. Pure-stdlib markdown engine (no third-party markdown library), four hand-tuned themes, self-contained HTML exports.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 122,327 | 79.5% |
| JavaScript | 15,439 | 10.0% |
| CSS | 11,765 | 7.6% |
| HTML | 4,374 | 2.8% |

## File tree

- Total entries indexed: **41** (31 files, 10 directories)

```
.gitignore  (434 B)
LICENSE  (1 KB)
README.md  (5 KB)
pyproject.toml  (2 KB)
requirements.txt  (161 B)
sample.md  (1 KB)
markdown_viewer/    [21 files]
  markdown_viewer/__init__.py
  markdown_viewer/__main__.py
  markdown_viewer/api/__init__.py
  markdown_viewer/api/main.py
  markdown_viewer/api/models.py
  markdown_viewer/api/routes.py
  markdown_viewer/api/workspace.py
  markdown_viewer/core/__init__.py
  markdown_viewer/core/analyzer.py
  markdown_viewer/core/exporter.py
  markdown_viewer/core/renderer.py
  markdown_viewer/core/themes.py
  markdown_viewer/desktop/__init__.py
  markdown_viewer/desktop/app.py
  markdown_viewer/desktop/editor.py
  ... and 6 more under markdown_viewer/
tests/    [4 files]
  tests/__init__.py
  tests/test_analyzer.py
  tests/test_renderer.py
  tests/test_themes_exporter.py
```

## README (verbatim)

# Markdown Viewer

A premium, minimalist markdown viewer that ships **three surfaces from one Python package**:

- a REST **API** (FastAPI) that renders, analyzes, and exports markdown,
- a **desktop app** built on tkinter with a live preview, outline, and themes,
- a **website** served by the API, with a live editor, command palette, and export.

The same engine drives all three, so what you see in the browser is exactly what you'll get from the desktop preview or a downloaded HTML page.

## Highlights

- **Pure-stdlib markdown engine** — no external markdown library. GFM-flavoured: tables, task lists, code blocks, autolinks, strikethrough, math passthrough (KaTeX), Mermaid passthrough.
- **Four hand-tuned themes** — Ivory (warm light), Graphite (refined dark), Sepia (paper), Midnight (deep blue). Tokens are shared across all three surfaces.
- **Self-contained HTML exports** — embedded fonts, theme CSS, optional KaTeX & Mermaid via CDN. Print-friendly.
- **Sandboxed file API** — `/api/files/*` routes are scoped to a single workspace directory.
- **Command palette** in the website (`Ctrl+K`).
- **44 unit tests** for the rendering and analysis core.

## Install

```bash
git clone https://github.com/ameyaborkar/markdown-viewer.git
cd markdown-viewer
python -m pip install -e ".[api]"
```

The desktop app and rendering core need no third-party dependencies — only the API surface needs FastAPI/uvicorn.

## Use

```bash
# Launch the desktop app (default action when no subcommand is given).
python -m markdown_viewer
python -m markdown_viewer desktop

# Start the REST API and bundled web client at http://127.0.0.1:8765/
python -m markdown_viewer api
python -m markdown_viewer web                 # same, opens a browser tab

# Render a markdown file to an HTML fragment on stdout.
python -m markdown_viewer render sample.md

# Build a polished, self-contained HTML page.
python -m markdown_viewer export sample.md -o out.html -t graphite
```

### Web client

`http://127.0.0.1:8765/` serves the bundled web app. Features:

- Live two-pane editor + preview, debounced through `/api/render`.
- Theme switcher (persisted), outline sidebar, splitter resize.
- File `Open`, `Save`, `Export HTML`.
- Command palette (`Ctrl+K`), keyboard shortcuts (`Ctrl+S`, `Ctrl+O`, `Ctrl+E`, `Ctrl+N`, `Ctrl+\`, `F11`).
- KaTeX inline + display math, Mermaid diagrams.

### Desktop app

The tkinter app mirrors the web layout: outline · editor · preview, with a status bar that tracks word count, reading time, and cursor position. Themes change the whole window; the preview is rendered into a styled `Text` widget so the typography stays consistent across platforms.

Keyboard: `Ctrl+N`, `Ctrl+O`, `Ctrl+S`, `Ctrl+Shift+S`, `Ctrl+E`, `Ctrl+F`, `Ctrl+\`, `Ctrl+Q`.

### REST API

`python -m markdown_viewer api` starts uvicorn on `127.0.0.1:8765`. Highlights:

| Method | Path                       | Purpose                                          |
|:------:|:---------------------------|:-------------------------------------------------|
| GET    | `/health`                  | Service heartbeat + version.                     |
| POST   | `/api/render`              | Markdown → HTML + headings + word count.         |
| POST   | `/api/analyze`             | Frontmatter, statistics, TOC tree.               |
| GET    | `/api/themes`              | List of theme palettes.                          |
| GET    | `/api/themes/{id}`         | One theme's palette.                             |
| GET    | `/api/themes.css`          | Ready-to-drop CSS variables for every theme.     |
| GET    | `/api/files/list?path=`    | Sandboxed listing of the workspace.              |
| GET    | `/api/files/read?path=`    | Sandboxed file read.                             |
| POST   | `/api/files/save`          | Sandboxed file write.                            |
| POST   | `/api/export/html`         | Build a self-contained HTML page (returns text). |
| POST   | `/api/export/html/download`| Same, with `Content-Disposition: attachment`.    |

The workspace defaults to the current working directory. Override it with `--workspace` or the `MARKDOWN_VIEWER_WORKSPACE` environment variable.

OpenAPI is at `/docs` and `/redoc`.

## Project layout

```
markdown_viewer/
  core/         renderer · themes · analyzer · exporter      (zero deps)
  api/          FastAPI app · routes · models · workspace
  desktop/      tkinter app · editor · preview · theme
  web/          static/css · static/js · templates/index.html
tests/          stdlib unittest suites for the core
sample.md       a small showcase document
```

## Tests

```bash
python -m unittest discover -s tests
```

## License

MIT — see [LICENSE](LICENSE).

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/markdown-viewer*
