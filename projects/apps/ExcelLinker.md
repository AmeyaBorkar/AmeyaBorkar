# ExcelLinker (ExcelFileLinker)

> A single-file Tkinter desktop GUI (Python) for loading multiple Excel/CSV workbooks and progressively joining their sheets into one merged dataset via column-based joins, then exporting to CSV/Excel. The shipped "current" version (`excel_linker.py`, v6.0) is a polished performance/UX shell — threaded loading, column-preview dialog, memory monitoring, project save/load, data-quality and validation reports — **but its core join engine is stubbed out** ("will be implemented in next phase"). The actually-working join logic (real `pd.merge` / `DataFrame.join` with inner/left/right/outer joins, undo, rebuild, smart join recommendations) lives only in the three archived earlier versions under `archive/`.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/ExcelFileLinker |
| Visibility | Public |
| Category | apps |
| Primary language(s) | Python (100%) |
| Local path | `C:\Users\ameya\.repo-cache\ExcelFileLinker` |
| Default branch | `main` |
| Lines of code (computed) | **5,874** total Python across 4 `.py` files (`excel_linker.py` 1,753 + `archive/v1` 981 + `archive/v2` 1,466 + `archive/v3` 1,674). README/requirements/gitignore add ~99 lines of non-code. |
| Source files (computed) | **4 Python files** (1 current entry point + 3 archived versions). No test files. |
| Key dependencies | `pandas>=1.5.0`, `openpyxl>=3.0.0` (Excel engine), `psutil>=5.9.0` (memory monitoring); stdlib `tkinter`, `threading`, `queue`, `json`, `gc`, `datetime` |
| License | README states **MIT**, but there is **no `LICENSE` file** committed (discrepancy). |
| Last commit | `8372374` — "Initial commit: ExcelLinker v6.0 with archive of previous versions" (2026-03-03). Single-commit repo. |

---

## What it actually is

ExcelLinker is a Windows/cross-platform **desktop GUI application** (one Python script, run with `python excel_linker.py`) for **cumulative, column-based joining of tabular data from Excel/CSV files**. The intended workflow:

1. Load one or more `.xlsx` / `.xls` files; every sheet is discovered and read into a `pandas.DataFrame`.
2. Pick a base file/sheet as the start of a "build" (the working dataset).
3. Repeatedly join additional sheets onto the working dataset by mapping a left column to a right column (a SQL-style join keyed on chosen columns).
4. Validate / inspect data quality, then export the merged result to CSV or Excel.

It is **not** a spreadsheet editor, **not** a formula/cross-reference linker between workbook cells, and **not** a web app. The "linking" is **relational joining of sheets via pandas merge**, conceptually a visual front-end for `pd.merge`.

**The most important code-derived fact:** in the current top-level `excel_linker.py` (v6.0), the join engine does not exist. The following methods are placeholders:

- `start_build()` → `messagebox.showinfo("Info", "Build functionality will be implemented in next phase")` (line 1615)
- `add_to_build()` → same "next phase" message (line 1619)
- `undo_last_join()` → "next phase" message (line 1633)
- `rebuild_dataset()` → `pass` (line 1637)
- `show_detailed_history()` → "next phase" message (line 1641)
- All UI-sync methods `update_sheet_combos`, `update_columns_tabs`, `update_build_status`, `update_dataset_preview`, `update_final_output_controls`, `update_build_history` → `pass` (lines 1591–1613)
- `create_main_content()` is a "placeholder for full implementation" that only builds a `Treeview` of loaded files/sheets (line 1525) — the join controls, sheet combo-boxes, column-selection tabs and dataset preview described in the README/About are **not built**.

A `grep` for `pd.merge` / `.merge(` in `excel_linker.py` returns **nothing**; the only real merge calls in the repo are in `archive/v1` and `archive/v2`, and `archive/v3` uses `DataFrame.join`. So v6.0 will launch, load files, show reports, save/load projects and export an existing working dataset — but a user can never actually *create* a working dataset through it, because Start Build / Add to Build are no-ops. Export only fires when `self.working_dataset is not None`, which in v6.0 can only happen via project-rebuild of a previously-saved build.

## Architecture & how it's structured (annotated tree)

```
ExcelFileLinker/
├── excel_linker.py        # v6.0 "current" entry point (1,753 lines). 5 classes + main().
│                          #   Core join engine STUBBED; loading/UI/reporting shell is real.
├── requirements.txt       # pandas>=1.5.0, openpyxl>=3.0.0, psutil>=5.9.0
├── README.md              # Feature claims (some ahead of v6.0 code), version-history table
├── .gitignore             # ignores *.xlsx/*.xls/*.csv (sample data) and *.json (projects)
└── archive/               # earlier, FULLY-WORKING versions kept for history
    ├── v1_excel_linker.py # 981 lines — single-sheet, inner-join only, real pd.merge
    ├── v2_excel_linker.py # 1,466 lines — multi-sheet + final-column designer, inner-join
    └── v3_excel_linker.py # 1,674 lines — + CSV input, 4 join types, smart join recommendations
```

No sample `.xlsx`/`.csv` data and no `*.json` project files are committed (excluded by `.gitignore`); none exist in the clone. There is **no `tests/` directory and no test files** anywhere.

The app is a **monolithic single-module** design: every version is one self-contained script with no shared/imported modules between them (the archive files are independent snapshots, not a package).

## Code walkthrough (the important parts) — module by module

`excel_linker.py` (v6.0) is organized as 5 classes plus `main()`:

- **`ThreadSafeProgress`** (line 24) — a lock-guarded progress holder (`current`/`total`/`message`) with an optional callback. Used to marshal progress from the loader thread.
- **`FileLoadResult`** (line 41) — a plain result container: `success`, `filename`, `error`, `data` (`{file_key: {sheet_name: DataFrame}}`), `memory_usage` (MB delta).
- **`ColumnPreviewDialog`** (line 51) — a `Toplevel` modal shown for "large" files. Renders one `Notebook` tab per sheet, each a scrollable list of `Checkbutton`s (one per column) plus a sample value. Buttons: Select All, Smart Selection, per-sheet All/None/Smart. `smart_select_sheet()` (line 187) deselects columns whose name contains any of `['unnamed','index','row_number','temp','debug','test']`. `get_selected_columns()` returns `{sheet: [chosen cols]}`.
- **`FileLoader`** (line 223) — the genuinely functional engine of v6.0:
  - `get_file_info()` (line 230) opens `pd.ExcelFile`, reads `nrows=5` per sheet for columns + samples, and estimates size as `rows * cols * 8 bytes`.
  - `should_show_column_preview()` (line 266) triggers the preview dialog if estimated size > **50 MB** or total columns > **50**.
  - `load_files_with_preview()` (line 276) splits files into preview-needed vs direct-load, runs the modal dialog per large file, then `start_threaded_loading()`.
  - `load_files_thread()` / `load_single_file()` (lines 330, 363) run on a daemon `threading.Thread`, read each sheet with `pd.read_excel(..., usecols=...)` honoring the column selection, measure RSS via `psutil`, and post results back to the Tk main thread with `root.after(0, ...)`.
- **`ExcelLinkerApp`** (line 413) — the main window and controller. Holds the data model:
  - `loaded_files: Dict[str, Dict[str, DataFrame]]` (file → sheet → df)
  - `file_paths`, `join_history: List[Dict]`, `working_dataset: Optional[DataFrame]`, `selected_columns`, `final_columns`.
  - Builds the menu bar, toolbar (emoji buttons), `Treeview` of files/sheets, and status bar (live memory label updated every 5 s via `root.after`).
  - Real, working features: file open/remove (`select_files`, `remove_selected_files`, with build-impact warnings), project save/load (`save_project`/`load_project` to/from JSON, version-tagged `'6.0'`), validation report (`run_validation_checks`, line 971), per-file data-quality report (`generate_file_quality_report`, line 1101), memory report with forced GC (lines 1194, 1271), plus several static help/about/settings dialogs.
  - Stubbed features: everything in the join/build/export pipeline (see above).
- **`main()`** (line 1729) — creates the `Tk()` root, instantiates the app, wires a `WM_DELETE_WINDOW` quit confirmation that cancels any in-flight loading, and `root.mainloop()`.

## The core logic — exactly how the Excel linking/processing works

Because v6.0's engine is stubbed, the **real linking algorithm must be read from the archive**. The model is identical across versions: a `join_history` list of step dicts and a single `working_dataset` DataFrame, with `rebuild_dataset()` able to replay all steps from scratch (this is what enables undo and project-restore).

**v1 / v2 (inner join via `pd.merge`).** `start_build()` copies the chosen base sheet's selected columns into `working_dataset`. `add_to_build()` then does (v2, lines 805–811; v1 lines 677–683):

```python
merged_df = pd.merge(
    self.working_dataset,
    join_df,
    left_on=left_col,
    right_on=right_col,
    how='inner'          # hard-coded — exact match only
)
```

Join type is **inner-only and hard-coded**; the UI shows a gray note "Using exact match (inner join)" rather than a selector. `undo_last_join()` pops the last history entry and calls `rebuild_dataset()`, which re-copies the base and **re-runs every `pd.merge`** from history. v1 is single-sheet-per-file; v2 adds multi-sheet loading (`pd.ExcelFile` → `read_excel(sheet_name=...)`) and an interactive final-output column designer.

**v3 (4 join types via index-based `DataFrame.join`).** This is the most complete version. `add_to_build()` (lines 1001–1007):

```python
join_df_indexed = join_df.set_index(right_col)
merged_df = self.working_dataset.join(
    join_df_indexed,
    on=left_col,
    how=join_type,       # user-selected
)
```

`join_type` comes from a `Combobox` with values `["inner","left","right","outer"]` (default `inner`). v3 also accepts **CSV input** (`pd.read_csv`) and adds **smart join recommendations**: `suggest_join_type()` (lines 480–516) computes key-overlap / match percentages between the two columns and returns advice (e.g. "LEFT JOIN recommended — only X% of left records will match"), surfaced live in a UI label via `update_join_recommendation()` (lines 733–802). It recommends a *join type* for already-selected columns; it does **not** auto-detect/auto-pair column names.

So conceptually: ExcelLinker treats each sheet as a relational table and lets the user chain SQL-style joins on matching key columns, accumulating into one wide dataset — exactly a GUI wrapper over pandas merge/join, with the full chain stored as replayable history.

## Interface — GUI or CLI, how a user drives it

Pure **Tkinter GUI**, no CLI/argparse. Launched with `python excel_linker.py`; `main()` opens a 1600×1000 window titled "Excel File Linker v6.0 — Enhanced Performance & Smart Loading".

Driven via:
- **Menu bar**: File / Build / Tools / Help cascades (line 894).
- **Emoji toolbar** (line 454): 📁 Open Files, 🗑️ Remove Files, 🔨 Start Build, ➕ Add to Build, 🔄 Reset Build, ↶ Undo, 📊 Export CSV, 📋 Export Excel, 💾 Save Project, 📂 Load Project, 🔄 Refresh.
- **Keyboard shortcuts** (line 438): `Ctrl+O` open, `Ctrl+S` export CSV, `Ctrl+R` reset, `Ctrl+Z` undo, `Del` remove, `F5` refresh. (Note: the Shortcuts help dialog also advertises `Ctrl+Shift+S/O` for save/load project, but those bindings are **not** actually wired in `setup_keyboard_shortcuts` — only menu labels mention them.)
- **Main content**: in v6.0 just a `Treeview` listing loaded files and their sheets (rows/cols), with a right-click context menu. The join controls / sheet combo-boxes / column-selection tabs / dataset preview that the README workflow assumes are **not rendered** in v6.0 (they exist in the archive versions).
- **Status bar**: live process memory (refreshed every 5 s), file/sheet counts, and a progress bar + Cancel button during threaded loads.

Input formats: `.xlsx`, `.xls` (v6.0 file picker); `.csv` input exists only in archived v3. Output formats: CSV (`to_csv(index=False)`) and Excel (`to_excel(index=False)`); archive versions write a richer multi-sheet workbook (Final_Output / Full_Dataset / Build_History / Column_Configuration) via `pd.ExcelWriter(engine='openpyxl')`, whereas v6.0's export is a single flat sheet.

## Tech stack & dependencies (which Excel library)

- **Excel library: pandas** for all reading/writing (`pd.ExcelFile`, `pd.read_excel`, `to_excel`, plus `read_csv`/`to_csv`), with **openpyxl** as the underlying `.xlsx` engine (declared in `requirements.txt`; archive versions name it explicitly as `engine='openpyxl'` in `ExcelWriter`). **No** `xlwings`, **no** `win32com`, **no** raw `openpyxl` cell manipulation — all I/O goes through pandas DataFrames.
- **GUI: tkinter / ttk** (stdlib) — `Notebook`, `Treeview`, `Combobox`, `Progressbar`, `Canvas`-based scroll regions, `Toplevel` dialogs. No PyQt/wx.
- **Concurrency: `threading` + `queue`**, with results marshalled back to Tk via `root.after(...)`.
- **`psutil`** for process RSS/VMS memory monitoring; stdlib **`gc`** for forced garbage collection; **`json`** for project files; **`datetime`** for timestamps. Type hints via `typing`.
- Targets Python **3.8+** (README); uses f-strings and standard typing.

## Build / run (actual commands)

```bash
# Clone
git clone https://github.com/AmeyaBorkar/ExcelFileLinker.git
cd ExcelFileLinker

# Install deps
pip install -r requirements.txt        # pandas, openpyxl, psutil

# Run the current (v6.0) GUI
python excel_linker.py

# Or run a fully-working older version with the real join engine:
python archive/v3_excel_linker.py      # 4 join types + CSV + smart recommendations
python archive/v2_excel_linker.py      # multi-sheet, inner-join, column designer
python archive/v1_excel_linker.py      # single-sheet, inner-join
```

No build step, no packaging (`setup.py`/`pyproject.toml` absent), no entry-point script, no Docker. Requires a desktop/display (Tkinter). The README's `cd ExcelLinker` is slightly wrong — the cloned directory is `ExcelFileLinker`.

## Status, completeness & notable gaps

- **The shipped v6.0 is a half-finished refactor.** Its loading pipeline, column-preview, memory tooling, reports and project save/load are real and reasonably polished, but the headline capability — actually building/joining a dataset — is replaced by "next phase" placeholder dialogs. A first-time user running `excel_linker.py` cannot produce a merged output. The working product is effectively the **archived v3** (or v2/v1).
- **No tests** of any kind (no `tests/`, no `pytest`, no CI workflow).
- **No `LICENSE` file** despite the README's MIT claim.
- **No persistence of settings**: `show_performance_settings` and `toggle_auto_save` only pop confirmation dialogs; their values are never stored or applied (`apply_settings` just shows a success message).
- **Single commit** in history — the entire project (current + archive) landed in one "Initial commit", so the "version history" is curated archive files rather than real git evolution.
- **Brittle keys**: file/sheet identity uses `f"{filename}::{sheet_name}"` string keys parsed by `split("::")`, and files are keyed by basename only, so two same-named files from different folders would collide.
- **Project rebuild timing**: `load_project` schedules `rebuild_dataset_from_project` via `root.after(2000, ...)` — a 2-second fixed delay assumed to outlast threaded loading, which is racy for large files (and moot in v6.0 since `rebuild_dataset` is `pass`).

## README vs. code

- **README implies a complete, working join app** ("loading, joining, and exporting", "Cumulative Build System… progressively join additional data"). For the **current `excel_linker.py` this is false** — Start Build / Add to Build / Undo / rebuild are stubs. The claim is true only for the archived versions. This is the single biggest README↔code discrepancy.
- README **version table** is accurate as a map: v6.0 = `excel_linker.py`; v5.0 = `archive/v3` (smart join recommendations, CSV, join-type selection — matches code); v4.0 = `archive/v2` (multi-sheet, column-selection, final-output customization — matches); v2.0 = `archive/v1` (cumulative builder, inner joins — matches). Note the README's "v5.0/v4.0/v2.0" labels don't line up with the file names `v3/v2/v1`, and there is no archived "v3.0/v1.0".
- README lists features as present in the app generally; in v6.0 specifically, **Build Validation, Data Quality Reports, Memory Monitoring, Project Save/Load, Threaded Loading and Smart Column Preview are genuinely implemented**, while **the join-based "Cumulative Build" and Export-of-a-built-dataset are not reachable** (export only works on an already-existing `working_dataset`).
- README: License **MIT** — but **no LICENSE file** exists in the repo.
- README install snippet says `cd ExcelLinker`; the actual repo/clone directory is **`ExcelFileLinker`**.
- The in-app About text claims "Supports Excel (.xlsx, .xls) and CSV files", but v6.0's file picker only offers `.xlsx`/`.xls`; **CSV input exists only in archived v3**.
- The Shortcuts help dialog lists `Ctrl+Shift+S`/`Ctrl+Shift+O` for project save/load, but these are **not bound** in code (only `Ctrl+O/S/R/Z`, `Del`, `F5` are).

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\.repo-cache\ExcelFileLinker. Source of truth: https://github.com/AmeyaBorkar/ExcelFileLinker*
