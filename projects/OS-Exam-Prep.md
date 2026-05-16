# OS Exam Prep

> A static single-page web app for revising Operating Systems for a VIT-style university final — bundles 20 mind maps, 6 cheat sheets, 60+ fully-solved model answers (including FCFS / SRTF / RR scheduling, Banker's algorithm, paging arithmetic, disk-scheduling head movements), all in one file you can open in any browser.

**Repository:** [`AmeyaBorkar/os-exam-prep`](https://github.com/AmeyaBorkar/os-exam-prep)  
**Category:** Study / Personal  
**Visibility:** Private  
**Primary language:** HTML  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-05-12  
**Last pushed:** 2026-05-12  
**Metadata updated:** 2026-05-12  
**Size (GitHub reported):** 309 KB  

---

## What it is (one-paragraph version)

A static single-page web app for revising Operating Systems for a VIT-style university final — bundles 20 mind maps, 6 cheat sheets, 60+ fully-solved model answers (including FCFS / SRTF / RR scheduling, Banker's algorithm, paging arithmetic, disk-scheduling head movements), all in one file you can open in any browser.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| HTML | 34,118 | 75.3% |
| Python | 10,709 | 23.6% |
| Batchfile | 470 | 1.0% |

## File tree

- Total entries indexed: **53** (40 files, 13 directories)

```
.gitignore  (268 B)
README.md  (5 KB)
build_mindmaps.py  (7 KB)
build_site.py  (4 KB)
content.js  (874 KB)
index.html  (33 KB)
open.bat  (245 B)
serve.bat  (225 B)
content/    [32 files]
  content/unit1/answers.md
  content/unit1/cheatsheet.md
  content/unit1/mindmaps/01-overview.md
  content/unit1/mindmaps/02-types-of-os.md
  content/unit1/mindmaps/03-services-and-syscalls.md
  content/unit2/answers.md
  content/unit2/cheatsheet.md
  content/unit2/mindmaps/01-overview.md
  content/unit2/mindmaps/02-process-states-lifecycle.md
  content/unit2/mindmaps/03-threads-and-multithreading.md
  content/unit2/mindmaps/04-concurrency-and-mutex.md
  content/unit3/answers.md
  content/unit3/cheatsheet.md
  content/unit3/mindmaps/01-overview.md
  content/unit3/mindmaps/02-algorithms-deep-dive.md
  ... and 17 more under content/
```

## README (verbatim)

# OS Exam Prep

A static single-page web app for revising **Operating Systems** for a VIT-style university final.

Bundles cheat sheets, model exam answers, and interactive mind maps for 6 units into one file you can open in any browser — no install, no server.

## What's inside

| Section | Per unit | Total |
|---|---|---|
| **Mind maps** | 3–4 top-down boxed maps | **20** |
| **Cheat sheets** | A) Visual sheet · B) Trigger points · C) Mnemonics · E) Rapid revision | **6** |
| **Model answers** | All question-bank items, 15-mark format, fully-solved numericals | **60+** |
| **Extras** | Master index · last-night cram card · cross-unit confusions | 1 |

Numericals covered (with steps + verified arithmetic):
- **Unit 3** — FCFS, SRTF, Round Robin (TQ=2)
- **Unit 4** — Banker's algorithm safety check (3 worked tables)
- **Unit 5** — Address translation (4 KB / 1 KB page sizes) · FIFO / LRU page replacement · thrashing
- **Unit 6** — FCFS / SSTF / SCAN / C-SCAN disk-scheduling (head movement totals)

## Getting started (clone & run)

> First time on this repo? Follow these steps. They take ~3 min on a clean machine.

### Prerequisites (one-time install)

| Tool | Why | Install |
|---|---|---|
| **Git** | clone the repo | https://git-scm.com/download (Mac: `brew install git`) |
| **Python 3** | local web server | https://www.python.org/downloads/ — tick **"Add to PATH"** during install |
| **GitHub CLI** | easy private-repo auth | https://cli.github.com — Windows: `winget install GitHub.cli` |

### Step 1 — Authenticate with GitHub (one-time)

Open a terminal (PowerShell on Windows, Terminal on macOS/Linux) and run:

```bash
gh auth login
```

When it asks:
- **GitHub.com** → Enter
- **HTTPS** → Enter
- **Authenticate Git with your credentials?** → **Y**
- **Login with a web browser** → Enter
- It prints an 8-char code (e.g. `ABCD-1234`) and opens `github.com/login/device`
- Paste the code → click **Authorize** → close the tab

### Step 2 — Clone the repo

```bash
gh repo clone AmeyaBorkar/os-exam-prep
```

*(equivalent: `git clone https://github.com/AmeyaBorkar/os-exam-prep.git`)*

### Step 3 — Enter the folder

```bash
cd os-exam-prep
```

### Step 4 — Start the local server

**Windows:** double-click `serve.bat`, OR run:
```powershell
python -m http.server 8000
```

**macOS / Linux:**
```bash
python3 -m http.server 8000
```

### Step 5 — Open in the browser

Visit **http://localhost:8000** — done.

### Pulling future updates

When the repo gets new content, run this inside the folder:
```bash
git pull
```
Then refresh the browser (**Ctrl+F5** / **Cmd+Shift+R** to bypass cache).

---

## Quick-launch (already cloned)

```bash
# Just open the file directly (no server needed)
double-click index.html        # Windows
open index.html                # macOS

# or with the bundled launchers:
./open.bat                     # opens via file://
./serve.bat                    # local HTTP server, then visit http://127.0.0.1:8000
```

Works offline after the first load — CDN libs cache automatically.

## Features

- **Light + dark theme** toggle (persists across reloads)
- **Collapsible sidebar** (`Ctrl+B` shortcut, or click the ☰)
- **Hash-based routing** so back/forward buttons work
- **Mind maps as static HTML** — no JS rendering library, no zooming weirdness, just bounded boxes with colored top borders and a top-down layout
- **Density toggle** on mind maps — cycle between compact / normal / large text

## Project layout

```
website/
├── index.html           SPA shell — UI, CSS, navigation
├── content.js           generated bundle (~870 KB) — all markdown + mindmap HTML
├── content/             source markdowns (this is what you edit)
│   ├── unit{1..6}/
│   │   ├── cheatsheet.md     sections A, B, C, E (the wall sheet)
│   │   ├── answers.md        section D (model exam answers)
│   │   └── mindmaps/
│   │       └── *.md          one markdown per mind map
├── build_mindmaps.py    parses mindmap markdowns → static HTML
├── build_site.py        bundles everything into content.js
├── open.bat             one-click launcher (file://)
├── serve.bat            local HTTP server (python -m http.server)
└── README.md            (this file)
```

## Rebuilding after editing content

```bash
python build_site.py
```

That re-reads every `.md` file under `content/`, renders the mind maps inline, and rewrites `content.js`. Reload the page (Ctrl+F5 to bypass cache).

## Tech stack

- **No framework** — vanilla HTML/CSS/JS
- **Mind maps** — Python-rendered static HTML; cards with colored top borders, nested bullets with dashed guide-rails, color-rotated per branch
- **Markdown** — [marked.js](https://marked.js.org/) via CDN (cheat sheets + answers only — mindmaps are pre-rendered)
- **Syntax highlighting** — [highlight.js](https://highlightjs.org/) via CDN
- **Total payload** — ~22 KB SPA shell + ~870 KB content bundle

## Question bank source

The 60+ model answers map 1-to-1 to the questions in the OS course question bank (10 per unit, with Unit 5 having 11). The question text appears verbatim as the heading for each answer (search `### Q1.` etc. inside any answers page).

## Licence

Personal study material. Not for redistribution as course content.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/os-exam-prep*
