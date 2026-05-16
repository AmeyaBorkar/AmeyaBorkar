# ExcelLinker

> Tkinter + Pandas desktop tool that lets non-technical users visually load, preview, select columns from, and progressively join multiple Excel files — with threaded loading and a project save-file format.

**Repository:** [`AmeyaBorkar/ExcelFileLinker`](https://github.com/AmeyaBorkar/ExcelFileLinker)  
**Category:** Desktop Utility  
**Visibility:** Public  
**Primary language:** Python  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-03-03  
**Last pushed:** 2026-03-03  
**Metadata updated:** 2026-03-03  
**Size (GitHub reported):** 41 KB  

---

## What it is (one-paragraph version)

Tkinter + Pandas desktop tool that lets non-technical users visually load, preview, select columns from, and progressively join multiple Excel files — with threaded loading and a project save-file format.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| Python | 258,970 | 100.0% |

## File tree

- Total entries indexed: **8** (7 files, 1 directories)

```
.gitignore  (315 B)
README.md  (2 KB)
excel_linker.py  (70 KB)
requirements.txt  (44 B)
archive/    [3 files]
  archive/v1_excel_linker.py
  archive/v2_excel_linker.py
  archive/v3_excel_linker.py
```

## README (verbatim)

# ExcelLinker

A desktop application for loading, joining, and exporting Excel files with a visual GUI. Built with Python, Tkinter, and Pandas.

## Features

- **Multi-file & Multi-sheet Loading** — Load multiple `.xlsx` / `.xls` files, auto-discovers all sheets
- **Smart Column Preview** — For large files, preview and select only the columns you need before loading
- **Threaded File Loading** — Background loading with progress tracking and cancel support
- **Cumulative Build System** — Start with a base file/sheet, then progressively join additional data
- **Data Quality Reports** — Analyze missing values, duplicates, data types per file
- **Memory Monitoring** — Real-time memory usage tracking with detailed reports and GC controls
- **Build Validation** — Validate current joins for data quality issues
- **Project Save/Load** — Save your project configuration (loaded files, joins, column selections) as JSON
- **Export** — Export final merged dataset to CSV or Excel (with build history sheet)
- **Keyboard Shortcuts** — `Ctrl+O` open, `Ctrl+S` export, `Ctrl+Z` undo, `F5` refresh, and more

## Requirements

- Python 3.8+
- Dependencies listed in `requirements.txt`

## Installation

```bash
# Clone the repository
git clone https://github.com/AmeyaBorkar/ExcelFileLinker.git
cd ExcelLinker

# Install dependencies
pip install -r requirements.txt
```

## Usage

```bash
python excel_linker.py
```

### Workflow

1. **Open Files** — Click 📁 or `Ctrl+O` to select Excel files
2. **Start Build** — Choose a base file/sheet and click 🔨 Start Build
3. **Add Joins** — Select a join file, map left/right columns, click ➕ Add to Build
4. **Validate** — Use Build → Validate or Tools → Data Quality Report
5. **Export** — Click 📊 Export CSV or 📋 Export Excel

## Version History

| Version | File | Key Changes |
|---------|------|-------------|
| v6.0 | `excel_linker.py` | Threaded loading, memory monitoring, project save/load, data validation |
| v5.0 | `archive/v3_excel_linker.py` | Smart join recommendations, CSV support, join type selection |
| v4.0 | `archive/v2_excel_linker.py` | Multi-sheet support, column selection tabs, final output customization |
| v2.0 | `archive/v1_excel_linker.py` | Initial version with cumulative builder, column selector, inner joins |

## License

MIT

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/ExcelFileLinker*
