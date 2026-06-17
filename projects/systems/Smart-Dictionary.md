# Smart Dictionary & Autocomplete Engine

> A C99 console (and optional GTK3 GUI) dictionary that builds **three tree structures in parallel** from the same word stream — an unbalanced Binary Search Tree, a self-balancing AVL tree, and a Knuth right-threaded binary tree — then lets you switch the "active" structure at runtime to compare search, insert, delete, prefix-autocomplete, and traversal behavior. It ships an ~90,000-word dictionary preprocessed from kaikki.org/Wiktionary, a frequency-plus-usage ranked autocomplete engine, file-based session persistence, and a built-in micro-benchmark harness timing BST vs AVL vs TBT on synthetic datasets of 500 / 2,000 / 5,000 words.

| Field | Value |
|-------|-------|
| Repository | [AmeyaBorkar/AdvanceDataStructuresSmartDictionary](https://github.com/AmeyaBorkar/AdvanceDataStructuresSmartDictionary) |
| Visibility | Public |
| Category | systems |
| Primary language(s) | C (C99) — CLI + GTK3 GUI; Python 3 (one-off data preprocessor); GNU Make |
| Local path | `C:\Users\ameya\Documents\smart_dictionary` |
| Default branch | `master` |
| Lines of code (computed) | **3,175** lines across `.c`/`.h` (2,749 in `.c`, 426 in `.h`); +296 Python, +94 Makefile. Core engine excluding the GUI ≈ **2,291** lines |
| Source files (computed) | **10 `.c` + 9 `.h` = 19 C source/header files**, +1 Python (`preprocess_jsonl.py`), +1 `Makefile` |
| Key components | `bst.c` (BST), `avl.c` (AVL), `tbt.c` (threaded BT), `autocomplete.c` (prefix engine), `benchmark.c` (timing harness), `loader.c` (file I/O + persistence), `main.c` (CLI), `gui_main.c` (GTK3 GUI) |
| License | **No `LICENSE` file in repo.** README states "released for educational use"; dictionary data derived from kaikki.org / Wiktionary (CC BY-SA 3.0) |
| Last commit | `1eaf50f` — "Version 3.0 Benchmarking, Persistence & Final Release (Phases 7-8)…" — 2026-03-10 |

> Note on working tree: the local clone has a large number of **uncommitted GTK3 runtime DLLs** (`libgtk-3-0.dll`, `libglib-2.0-0.dll`, etc.) plus built `.exe`/`.o` artifacts present but git-ignored. This doc is written against the source as it sits on disk.

---

## What it actually is

A single-author academic "Advanced Data Structures" project. Every dictionary entry is a fixed-size `WordRecord` (see `dictionary.h`), and the program inserts **the same record into all three trees at once** on every load/insert/delete. A global `g_active_tree` flag (`1=BST, 2=AVL, 3=TBT`) selects which tree services the next search / autocomplete / display, so the user can directly observe how the same operation behaves across structures that hold identical data.

The three structures actually implemented (verified by reading the source):

- **BST** (`bst.c`/`bst.h`) — unbalanced binary search tree. Notably engineered to be crash-safe on degenerate input: **insert/search are iterative**, and inorder/preorder/count use **Morris threading (O(1) extra space)**, while `bst_free` uses a **tree-to-vine (DSW vine phase)** iterative teardown. (This contradicts the README, which calls BST inorder "Recursive" — see README vs. code.)
- **AVL** (`avl.c`/`avl.h`) — self-balancing BST with a `height` field per node and the four standard rotations (LL/LR/RR/RL). Insert/delete/search/traversal/free are **genuinely recursive**.
- **TBT** (`tbt.c`/`tbt.h`) — a **right-threaded binary tree with a Knuth header/sentinel node**. NULL child slots are repurposed as inorder predecessor/successor "threads" so inorder traversal needs **no stack and no recursion**. Thread state is tracked with two full `int` flags per node (`lthread`, `rthread`), deliberately not bitfields (comment cites C99/MinGW bitfield UB).

There is **no trie**. Autocomplete is implemented as ordered-tree range collection, not a prefix tree.

Two front-ends are built from the same shared object files: a text menu (`main.c` → `smart_dict.exe`) and a GTK3 desktop app (`gui_main.c` → `smart_dict_gui.exe`). The GUI is real (884 lines, builds a toolbar / left result list / right detail panel, live autocomplete on entry change, insert/delete dialogs, file chooser, and a benchmark dialog that captures the CLI benchmark's stdout) but is **not mentioned anywhere in the README**.

---

## Architecture & how it's structured (annotated tree)

```
smart_dictionary/
├── config.h            Global constants: MAX_WORD_LEN=64, MAX_MEANING_LEN=512,
│                        MAX_WORDS=100000, TOP_K_DEFAULT=10, data file paths,
│                        APP_VERSION "1.0.0"  (note: repo is tagged "Version 3.0")
├── dictionary.{c,h}    WordRecord struct (the payload shared by all 3 trees)
│                        + init / compare (case-insensitive) / print helpers
├── utils.{c,h}         ASCII str_tolower/upper, str_trim*, str_safe_copy (strlcpy-
│                        style), starts_with[_ci], input_read_line (fgets+trim),
│                        print_separator/header
│
├── bst.{c,h}           Unbalanced BST. Iterative insert/search; Morris inorder/
│                        preorder/count; DSW-vine iterative free; iterative height
├── avl.{c,h}           AVL tree. Recursive insert/delete/search; rotate_left/right,
│                        rebalance; height field; recursive inorder/free/count
├── tbt.{c,h}           Right-threaded BT w/ Knuth header. Iterative insert/search/
│                        inorder/count; inorder_successor via threads; delete = rebuild
│
├── loader.{c,h}        load_words (1–5 field pipe parser), load_frequencies (CSV
│                        word,score), save_custom_words (preorder snapshot)
├── autocomplete.{c,h}  autocomplete_bst/avl (recursive collect + qsort),
│                        autocomplete_tbt (iterative lower-bound + thread walk),
│                        record_selection (bumps user_select_count in all 3 trees)
├── benchmark.{c,h}     benchmark_run_all → bench_one(n) for n∈{500,2000,5000}:
│                        synthetic "wd%05d" words, Fisher-Yates seed=42, times
│                        insert/search×1000/traverse, prints BST/AVL height
│
├── main.c              CLI: 9-option menu, global g_bst/avl/tbt roots + g_active_tree,
│                        auto-load on start, auto-save on exit
├── gui_main.c          GTK3 GUI (smart_dict_gui.exe) — same shared backend
│
├── preprocess_jsonl.py kaikki.org JSONL → words.txt (3-field), Python one-off
├── Makefile            builds smart_dict.exe (CLI) AND smart_dict_gui.exe (GTK3)
│
├── data/
│   ├── words.txt              90,005 lines (~90,000 entries + header comments),
│   │                          format: word|pos|meaning   (~6.1 MB)
│   ├── words_original.txt     99 lines — small curated test set
│   ├── word_freq.txt          97 lines — "word,score" CSV (mostly comments)
│   ├── custom_words.txt        runtime session snapshot, 5-field
│   │                          (word|pos|meaning|freq|picks); git-ignored
│   ├── words_processed.txt    preprocessor output (git-ignored, dup of words.txt)
│   └── kaikki.org-...jsonl    2.7 GB raw source (git-ignored, not in repo)
├── docs/report.md      EMPTY (0 bytes)
├── versions/{v1.0,v2.0,v3.0}/ snapshots of earlier phases (v1 = BST only;
│                              v2 adds avl/tbt/autocomplete; v3 = current)
├── CONTRIBUTIONS.md, README.md, Smart_Dictionary_Project_Document.docx
└── *.dll               ~45 uncommitted GTK3 runtime DLLs (for the GUI exe)
```

Data flow: `loader` reads files → builds `WordRecord`s → `bst_insert` / `avl_insert` / `tbt_insert` populate all three trees in lockstep (`loader.c:109-111`). `main.c` keeps `g_bst_root`, `g_avl_root`, `g_tbt_header` as globals and dispatches each menu action to the active tree.

---

## Code walkthrough (the important parts)

### BST — `bst.c` (255 lines)
The interesting thing here is defensive engineering against the classic "sorted input → fully skewed tree → recursion blows the stack at ~90k nodes" failure mode:
- `bst_insert` / `bst_search` are **iterative** (pointer-to-pointer cursor; `bst.c:76-112`).
- `bst_inorder`, `bst_preorder`, `bst_count` use **Morris traversal** — temporarily threading right-NULL pointers, restoring them, O(1) extra space (`bst.c:126-176, 233-255`).
- `bst_free` is the **DSW vine phase**: right-rotate every left child into a right-linked list, then free in one pass (`bst.c:184-201`).
- `bst_height` is iterative DFS with a fixed 128-frame stack (`bst.c:210-227`).
- `bst_delete` is the only recursive op, with the standard leaf / one-child / two-children-via-inorder-successor cases (`bst.c:22-54`).

### AVL — `avl.c` (170 lines)
Textbook AVL. `update_height`, `rotate_right`/`rotate_left`, and a `rebalance` that dispatches LL/LR/RR/RL by balance factor (`avl.c:36-60`). `avl_insert`/`avl_delete` are recursive and **return the new subtree root**, so callers must reassign (`g_avl_root = avl_insert(g_avl_root, &rec)` — enforced everywhere). Insert/delete lowercase the word before recursing. `avl_inorder`/`avl_free`/`avl_count` are recursive.

### Threaded Binary Tree — `tbt.c` (228 lines)
A right-threaded BT with a header sentinel (`tbt_create_header`, `tbt.c:39-48`): empty tree → `header->left = header` with `lthread=1`. `tbt_insert` does a BST-style descent respecting thread flags, then wires the new node's predecessor/successor threads from the parent (`tbt.c:61-112`). `tbt_inorder_successor` (`tbt.c:131-139`) is the crux: if `rthread`, the right pointer *is* the successor; else go right once and walk left to the subtree minimum. `tbt_inorder`/`tbt_count` walk from leftmost to header purely via threads, no stack. **Delete is a full rebuild** (`tbt.c:156-200`): copy all non-target records to a temp array, free all nodes iteratively, reset the header, re-insert — the comment notes this keeps thread logic simple at the cost of O(n) per delete.

### Autocomplete engine — `autocomplete.c` (152 lines)
`composite_score = frequency_score + 10 * user_select_count` (`autocomplete.c:14`). Two strategies:
- **BST/AVL**: recursive `bst_collect`/`avl_collect` with prefix pruning via `strncmp(node->word, prefix, plen)` — go left if past the prefix, right if before, collect + recurse both if it matches (`autocomplete.c:29-62`). Collected into a 512-entry buffer.
- **TBT**: fully iterative. First BST-descend to the **lower bound** (leftmost node whose first `plen` chars are ≥ prefix), then walk forward via `tbt_inorder_successor`, breaking as soon as `strncmp > 0` (`autocomplete.c:94-134`).
- Both then `qsort` candidates by composite score descending and copy the top `top_k` into the caller's array.
- `autocomplete_record_selection` searches all three trees for the chosen word and increments `user_select_count` in each (`autocomplete.c:142-152`), so picks personalize future rankings across structures.

### Benchmark harness — `benchmark.c` (189 lines)
This is what the project's headline "benchmarks BST vs AVL vs threaded binary tree" actually measures — and it's worth being precise: **it does NOT benchmark on the ~90k real words.** `gen_words` (`benchmark.c:34-53`) synthesizes `n` words formatted `wd%05d` (`wd00001`…), assigns `frequency_score = 1 + (i%100)`, then **Fisher-Yates shuffles with a fixed `srand(42)`** for reproducible insertion order. For each `n ∈ {500, 2000, 5000}` (`benchmark.c:14`), `bench_one` times:
1. Bulk insert into each tree.
2. Tree height (BST and AVL only; TBT height printed as `"n/a"`).
3. `BENCH_SEARCH_REPS = 1000` random searches (`srand(99)` re-seeded identically for each tree).
4. One full inorder traversal with a no-op callback.

Results print as three per-size blocks of four rows (Bulk insert ms / Tree height / Search x1000 ms / Traverse full ms) plus an explanatory notes footer. Because insertion order is shuffled (not sorted), the BST does **not** degenerate here — so the harness illustrates AVL/TBT staying low-height but does not exercise BST's worst case.

### I/O & persistence — `loader.c` (187 lines)
- `load_words`: per-line parser auto-detecting **1 to 5 pipe fields** — `word`, `word|pos`, `word|pos|meaning`, up to `word|pos|meaning|freq|picks` (`loader.c:42-117`). Skips blank/`#` lines and over-length words; inserts each record into all three trees.
- `load_frequencies`: CSV `word,score`, clamps to `[FREQ_SCORE_DEFAULT, FREQ_SCORE_MAX]`, updates the matching node in all three trees (`loader.c:119-165`).
- `save_custom_words`: writes the BST in **preorder** (not inorder) so reloading recreates the *same tree shape* rather than a sorted sequence that would skew BST/TBT — the comment explicitly calls out the O(n²)/stack-overflow trap this avoids (`loader.c:167-187`).

`main.c` auto-loads on startup (tries `custom_words.txt` first, falls back to `words.txt` + `word_freq.txt`) and auto-saves to `custom_words.txt` on exit.

---

## Data structures & algorithms

| Structure | File | Operations (impl. style) | Complexity / what's measured |
|-----------|------|--------------------------|------------------------------|
| **Binary Search Tree** | `bst.c` | insert/search iterative; inorder/preorder/count via **Morris**; free via **DSW vine**; height iterative DFS; delete recursive | Search/insert O(log n) avg, **O(n) worst** (degenerate). All bulk ops O(n) time, **O(1) extra space** (no recursion → safe on 90k skewed trees) |
| **AVL Tree** | `avl.c` | insert/delete/search/inorder/free **recursive**; LL/LR/RR/RL rotations | Search/insert/delete **O(log n) guaranteed**; height kept ≤ ~1.44·log₂n |
| **Threaded Binary Tree (right-threaded, Knuth header)** | `tbt.c` | insert/search/inorder/count **iterative via threads**; delete = **collect→free→rebuild** | Search/insert O(log n) avg; inorder traversal **O(n), no stack/recursion**; **delete O(n)** (full rebuild) |
| **Trie** | — | **not implemented** | Autocomplete uses tree range queries instead |
| **Autocomplete (BST/AVL)** | `autocomplete.c` | recursive prefix-pruned collect + qsort top-K | ≈ O(log n + m) to gather m matches, + O(m log m) sort |
| **Autocomplete (TBT)** | `autocomplete.c` | iterative lower-bound descent + forward thread walk + qsort | O(log n) to lower bound, O(m) walk, O(m log m) sort |

**Measured results:** The repo contains **no captured benchmark output** — `docs/report.md` is empty (0 bytes), and the only numbers labeled "Benchmark Sample Output" live in the README. Those README numbers (e.g. BST height 18/28/37 vs AVL 9/11/13 at 500/2000/5000) are **illustrative and do not match the actual `benchmark.c` print format** (which has no "=== Benchmark Results ===" combined table and prints TBT height as `n/a`). Treat them as hand-written examples, not recorded measurements. The qualitative conclusion the code's footer asserts — AVL/TBT stay logarithmic in height; BST degrades toward O(n) on sorted input — is correct in principle but, as noted, the harness's shuffled input does not actually trigger BST degeneration.

---

## Tech stack & build system

- **Language:** C99 (`-std=c99`), ASCII-only string handling (no locale `ctype`).
- **Compiler:** `gcc` (MinGW-w64 on Windows) with `-Wall -Wextra -Wpedantic -std=c99 -g`.
- **GUI:** GTK3 via `pkg-config gtk+-3.0`, linked `-mwindows`; ~45 GTK runtime DLLs sit in the repo dir for distribution.
- **Build:** GNU Make (`Makefile`, 94 lines). Compiles 8 shared `.c` files into `.o`, then links two executables (`smart_dict.exe` from `main.o`, `smart_dict_gui.exe` from `gui_main.o`). Targets: `all`, `cli`, `gui`, `run`, `run-gui`, `clean`, `rebuild`. (`clean` uses Windows `del`.)
- **Data tooling:** `preprocess_jsonl.py` (Python 3, 296 lines) — converts the 2.7 GB kaikki.org JSONL into the 3-field `words.txt`, capping at 90,000 entries and enforcing `config.h` length limits.
- **Dataset:** `data/words.txt` ≈ 90,000 entries (`word|pos|meaning`), ~6.1 MB. The raw JSONL and processed dup are git-ignored.

---

## Build / run / test (actual commands)

From MSYS2 / Git Bash on Windows, with MinGW + GTK3 on PATH:

```bash
# install toolchain (MSYS2)
pacman -S mingw-w64-x86_64-gcc make mingw-w64-x86_64-gtk3
export PATH="/c/msys64/mingw64/bin:$PATH"

make            # build BOTH smart_dict.exe (CLI) and smart_dict_gui.exe (GTK3)
make cli        # CLI only
make gui        # GUI only
make run        # build + run the CLI
make run-gui    # build + run the GUI
make clean      # remove .o + .exe (uses Windows `del`)
make rebuild    # clean then build both

# run directly (must be launched from project root so data/ is found)
./smart_dict.exe
```

CLI menu (from `main.c`): 1 Search, 2 Insert, 3 Delete, 4 Autocomplete, 5 Display all (sorted), 6 Load dictionary (file/test data), 7 Switch active tree, 8 Run benchmark, 9 About, 0 Exit. Default active tree is **BST**. The dictionary auto-loads on start and auto-saves to `data/custom_words.txt` on exit.

**Tests:** there is **no automated test suite** — no test runner, no CI config. "Testing" is the interactive benchmark (option 8) plus the 15 hardcoded `TEST_WORDS` in `main.c` (loaded when `words.txt` is missing).

---

## Status, completeness & notable gaps

**Complete / working:**
- All three tree structures fully implemented with insert/search/delete/traversal/free.
- Autocomplete with three per-structure strategies and cross-tree personalization.
- 1–5 field file loader, frequency loader, preorder session persistence.
- Benchmark harness producing a formatted comparison table.
- Two front-ends (CLI + GTK3 GUI) sharing one backend.
- Real ~90k-word dataset committed.

**Gaps / rough edges:**
- **`docs/report.md` is empty** — no benchmark results are actually recorded anywhere in the repo.
- **Benchmark uses synthetic shuffled data (500/2000/5000), not the 90k real words**, and never exercises BST's worst case despite the project framing.
- **No trie**, despite "autocomplete engine" framing (it's tree range queries).
- **No tests, no CI.**
- `config.h` still says `APP_VERSION "1.0.0"` while the repo is tagged "Version 3.0".
- **No `LICENSE` file** (README claims educational use only).
- Working tree is messy: ~45 uncommitted GTK DLLs and build artifacts; `versions/` duplicates earlier phases of the same code in-tree.
- `tbt_delete`'s O(n) rebuild and the dangling left-thread pointers it leaves (intentionally, never followed) are correct but fragile if traversal logic ever changes.

---

## README vs. code

The pre-existing doc (`projects/Smart-Dictionary-and-Autocomplete-Engine.md`) was generated from the **GitHub API + verbatim README**, not from source — so it inherits every README inaccuracy. Discrepancies found by reading the code:

1. **Inorder traversal "Recursive" for BST — WRONG.** README's comparison table lists BST inorder as Recursive. In `bst.c`, inorder/preorder/count use **Morris (iterative, O(1) space)**; only `bst_delete` recurses. AVL inorder *is* recursive (README correct there); TBT iterative (correct).
2. **TBT extra memory "two thread-flag bits" — IMPRECISE.** Code uses two full `int` flags per node (`tbt.h:34-35`), deliberately *not* bitfields (comment cites C99/MinGW UB). So it's two ints, not two bits.
3. **CLI menu in README does not match `main.c`.** README shows "Active tree: AVL", option 9 = "Save session", 0 = "About", "Q. Quit". Actual: default tree **BST**, 8 = Benchmark, 9 = About, 0 = Exit; there is **no manual Save option and no Q** — saving is automatic on exit.
4. **GUI is undocumented.** The Makefile builds a full **GTK3 app** (`gui_main.c`, 884 lines, `smart_dict_gui.exe`); the README is CLI-only and never mentions it.
5. **"Benchmark Sample Output" in README is fabricated/illustrative.** Its `=== Benchmark Results ===` combined table does not match `benchmark.c`'s actual output (per-size 4-row blocks; TBT height shown as `n/a`). No real results are stored in the repo (`docs/report.md` empty).
6. **"O(log n + k) on average … BST-pruning" for autocomplete — partially loose.** The collector does prune by `strncmp`, but on a prefix match it recurses *both* subtrees, so cost is dominated by the number of matches; the characterization is roughly right but optimistic for short prefixes.
7. **Stats are stale:** old doc reports "266,512 C bytes / 86 entries" from GitHub metadata; the source actually totals **3,175 lines of C across 19 files** (computed here). Category was "Data Structures (educational)"; this doc files it under **systems** per the catalog.

Everything else in the README (WordRecord layout, composite-score formula, Knuth header description, single-allocation-per-node, AVL-root-capture requirement, TBT delete-by-rebuild, preorder-save rationale) **matches the code accurately.**

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\smart_dictionary. Source of truth: https://github.com/AmeyaBorkar/AdvanceDataStructuresSmartDictionary*
