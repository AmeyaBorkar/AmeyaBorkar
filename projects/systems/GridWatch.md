# GridWatch

> GridWatch is an emergency-dispatch simulation of a grid city, written as a frontend-agnostic C11 library (`libdispatch`) plus a Win32 terminal UI and a Flask/ctypes web UI. It models ambulances, fire trucks, and police responding to randomly-spawned incidents on a road graph, and it wires roughly two dozen advanced data structures (Fibonacci heap, suffix array, BK-tree, DSU, segment tree, succinct bitvector, AVL/RB/splay/B+/threaded trees, skip list, treap, quadtree/kd/R/range/BSP/interval trees, DAWG, trie, Huffman, double-ended PQ, etc.) into the simulation so that the live HUD metrics are each computed by a different DS. It was built around an "Advanced Data Structures" course syllabus, and the code is deliberately split so the same compiled library drives all three frontends.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/GridWatch |
| **Visibility** | Public |
| **Category** | systems |
| **Primary language(s)** | C (C11) for the library/TUI; Python (Flask + ctypes) and vanilla JS/HTML/CSS for the web UI |
| **Local path** | `C:\Users\ameya\Documents\ADScp` |
| **Default branch** | `main` |
| **Lines of code (computed)** | C: 8,201 lines across 49 `.c` files; C headers: 948 lines across 11 `.h` files; Python: 1,688 lines across 4 `.py` files (incl. `make_ppt.py`); JS: 337 lines; HTML+CSS: 398 lines |
| **Source files (computed)** | 49 `.c`, 11 `.h`, 4 `.py`, 1 `.js`, 1 `.html`, 1 `.css` (excluding `build/` and `.git/`) |
| **Key components** | `src/ds/**` (30 DS implementations), `src/sim/*` (simulation core, 8 files / 1,290 LOC), `src/tui/*` (terminal UI, 4 files / 900 LOC), `web/` (Flask server + ctypes wrapper + static frontend), `tests/*` (7 per-module test binaries / 1,058 LOC) |
| **License** | None found — there is no `LICENSE` file in the repository |
| **Last commit** | `afb54e2` — 2026-05-06 — "docs: annotate skip list, treap, rng, kd/r/range/bsp trees, metrics, and web wrapper" (26 commits total). Working tree has 4 *untracked* files (`Group7.pdf`, `Group7.pptx`, `make_ppt.py`, `paper.tex`); no tracked files are modified |

---

## What it actually is

GridWatch (internally and in the README often called the "Emergency Dispatch Simulator") is a headless C simulation library with two thin frontends. The simulation builds a rectangular grid city of intersections and roads, places three stations (ambulance/fire/police), spawns random incidents over time, and dispatches the nearest appropriate idle unit to each incident along a shortest road path. Every step is instrumented so that on-screen counters (PQ operations, dispatch calls, Huffman compression ratio, suffix-array size, connected road components, etc.) are produced by real data structures rather than mocked numbers.

The architecture is the textbook **opaque-handle / POD-snapshot** pattern (`include/dispatch/sim.h`):

- All simulation state lives behind an opaque `sim_t*`; callers never see internal structs.
- UIs read state by requesting flat POD "view" snapshots (`sim_node_view_t`, `sim_unit_view_t`, `sim_incident_view_t`, `sim_metrics_t`, …) that copy cleanly across any FFI boundary.
- The same compiled library serves a static archive (`libdispatch.a`, linked into the TUI) and a shared DLL (`libdispatch.dll`, loaded by Python `ctypes`). The `DS_API` macro in `common.h` switches between `dllexport`/`dllimport`/nothing.

The "advanced data structures" theme is the project's whole reason for existing: the README maps each course-syllabus DS to a file and to a simulation feature. **Not every DS is actually exercised by the running simulation** — many are built/maintained as live indices but their query side is only exercised by the unit tests (see *Data structures inventory* and *Status* below). This is the most important nuance the README understates.

This is a course/portfolio project (note the `Group7.pdf`, `Group7.pptx`, `paper.tex`, and `make_ppt.py` artifacts at the repo root — a report, slide deck, paper, and a deck generator), not a production dispatch system.

---

## Architecture & how it's structured

Three layers, strictly separated; the dependency arrow points downward only.

```
                 ┌───────────────────────────┐        ┌──────────────────────────────┐
                 │  TUI  (src/tui/*.c)        │        │  Web UI (web/)               │
                 │  Win32 ANSI terminal app   │        │  Flask server + ctypes +     │
                 │  links libdispatch.a       │        │  static JS — loads .dll      │
                 └─────────────┬─────────────┘        └───────────────┬──────────────┘
                               │  both include ONLY dispatch/sim.h     │
                               │  (the public ABI contract)            │
                               └───────────────┬───────────────────────┘
                                               ▼
                          ┌────────────────────────────────────────┐
                          │  sim core  (src/sim/*.c)                │
                          │  city graph · stations/units/incidents  │
                          │  dispatcher · routing · log · metrics   │
                          │  owns one instance of every DS below    │
                          └────────────────────┬───────────────────┘
                                               ▼
   ┌──────────────────────────────────────────────────────────────────────────────────┐
   │  libdispatch DS modules  (src/ds/**)                                               │
   │  heaps/  trees/  strings/  randomized/  spatial/  misc/                            │
   └──────────────────────────────────────────────────────────────────────────────────┘
```

### Annotated source tree

```
include/dispatch/
  common.h        ds_key_t/ds_val_t, ds_status_t, ds_entry_t, DS_API export macro
  heaps.h         Fibonacci/binomial/leftist/skew/pairing heaps + DEPQ
  trees.h         AVL, RB, splay, B+, threaded BST + Huffman coder
  strings.h       trie, compressed radix trie, suffix array, DAWG, BK-tree fuzzy
  randomized.h    skip list, treap, rnd_seed (splitmix64)
  spatial.h       quadtree, kd-tree, R-tree, segment tree, interval tree, range tree, BSP
  misc.h          DSU (union-find), persistent list, succinct bitvector (rank/select)
  sim.h           THE public simulation ABI (opaque sim_t* + POD views)
  dispatch.h      umbrella header that includes all of the above

src/ds/
  heaps/    binom.c depq.c fib.c leftist.c pairing.c skew.c       (889 LOC)
  trees/    avl.c bplus.c huffman.c rb.c splay.c threaded.c       (1,344 LOC)
  strings/  crtrie.c dawg.c fuzzy.c sa.c trie.c                   (884 LOC)
  randomized/ rng.c skip.c treap.c                                (385 LOC)
  spatial/  bsp.c itree.c kd.c quad.c range.c rtree.c seg.c       (1,086 LOC)
  misc/     bv.c dsu.c plist.c                                    (365 LOC)

src/sim/                                                          (1,290 LOC)
  sim_internal.h  the concrete `struct sim` (every DS pointer lives here)
  sim.c           lifecycle (create/destroy), the tick loop, snapshot exporters, prng
  city.c          grid + road generation, CSR adjacency build, DSU connectivity sweep
  entities.c      stations/units/incidents, spawn fan-out, dispatch, unit motion FSM
  dispatcher.c    DEPQ-driven severity scheduler (drains pending each tick)
  routing.c       Dijkstra over CSR graph, backed by the Fibonacci heap
  log.c           rolling 64 KiB radio log + periodic suffix-array / Huffman rebuilds
  metrics.c       per-tick segment-tree bucket roll + bitvector rebuild
  search.c        wires trie / crtrie / DAWG / BK-tree dictionaries at startup

src/tui/                                                          (900 LOC)
  tui.h           shared TUI state struct
  main.c          fixed-30-FPS game loop (input → sim_tick → render)
  render.c        ANSI double-buffered map + HUD + "DS in action" panel + help
  input.c         Win32 raw keyboard handling
  console.c       Win32 console / ANSI setup

tests/            test_heaps.c test_trees.c test_strings.c test_randomized.c
                  test_spatial.c test_misc.c test_sim.c                (1,058 LOC)

web/
  dispatch_py/__init__.py  ctypes wrapper: mirrors every view struct + binds every export
  server.py                Flask app: SimRunner owns the Sim + 30 Hz background ticker
  static/index.html, app.js, app.css   zero-dependency browser frontend
  test_wrapper.py          self-test for the ctypes wrapper
  run.sh / run.bat         build the DLL if missing, then start the server

Makefile, build.sh         two equivalent build paths (GNU make and portable bash)
docs/CONCEPTS.md           plain-English tour of every on-screen metric
README.md, CONTRIBUTIONS.md
Group7.pdf/.pptx, paper.tex, make_ppt.py   course report / slides / paper (untracked)
```

### Data flow (one frame)

1. **Frontend** polls input / receives an HTTP request and calls `sim_tick(S, dt)`.
2. `sim_tick` (in `sim.c`) runs four phases: (1) `sim_dispatch_pending` drains the severity DEPQ; (2) `sim_tick_units` advances every unit's motion FSM; (3) spawns new incidents per `spawn_rate`; (4) rebuilds the idle-unit quadtree and runs `sim_metrics_tick`.
3. Dispatch (`sim_try_dispatch`) queries the **quadtree** for nearest idle candidates, resolves each candidate id→unit via the **AVL** tree, then runs **Dijkstra** (Fibonacci-heap PQ) on the **CSR** road graph to pick the lowest-travel-time unit; on commit it updates many secondary indices.
4. Each log line written by the sim flows into the rolling radio-log buffer; periodically the **suffix array** and **Huffman** code are rebuilt over it.
5. The frontend reads back POD snapshots (`sim_units`, `sim_incidents`, `sim_metrics`, `sim_log_tail`, …) and renders. The web server marshals those into JSON at `/state`.

---

## Code walkthrough (the important parts)

### `src/sim/sim.c` — lifecycle, tick loop, snapshots
`sim_create(rows, cols, seed)` allocates the `struct sim`, seeds a per-instance splitmix64 PRNG, builds the city, then **constructs an instance of essentially every DS in the library** and stores the pointers on the struct. There's a notable "warm-up" block that pushes/pops one element into each of the binomial/leftist/skew/pairing heaps — these meldable heaps exist for demonstration and are otherwise unused at runtime. `sim_tick` is the 4-phase orchestrator described above (no-op when paused or `dt<=0`). The bottom half of the file is the snapshot exporters that shallow-copy internal records into FFI-safe `*_view_t` POD structs, plus `sim_metrics` (which rebuilds a fresh Huffman tree on demand to report the current compression ratio) and `sim_log_count` (suffix-array substring count, with a naive O(n·m) fallback when no SA has been built yet).

### `src/sim/city.c` — graph + connectivity
Builds an R×C grid of intersection nodes (each given a "Street & Street" name from a 24-entry pool) and the horizontal+vertical roads between them, with **every 7th road blocked** (`counter % 7 == 6`). It then constructs a two-pass **CSR (compressed-sparse-row) adjacency** (count degrees → prefix-sum → scatter), which is the cache-friendly representation Dijkstra walks. Finally it runs a **DSU (union-find)** sweep that unions endpoints of all *unblocked* roads, records `road_components`, finds the largest component, and flags every node's `in_main_cc` so spawns/stations only land on the drivable network.

### `src/sim/routing.c` — Dijkstra + Fibonacci heap
`sim_dijkstra(S, src, tgt, dist, prev)` is a genuine single-source Dijkstra using `heap_fib_*` as its priority queue. It keeps a `handles[]` array (node id → Fib-heap node pointer) so it can call **`heap_fib_decrease_key`** in place (O(1) amortized) when relaxing an edge to a node already in the queue, rather than pushing duplicates. It tallies `pq_operations` on every push/pop/decrease-key and `dispatch_calls` once per run — these are the live HUD counters. `sim_reconstruct_path` walks the `prev[]` predecessor array backward and reverses it.

### `src/sim/dispatcher.c` — severity scheduling
`sim_dispatch_pending` drains up to 8 incidents per tick from the **double-ended priority queue**, always via `depq_pop_max` so the highest-severity call is handled first. If no suitable unit is free, the incident is re-queued at the same severity and the loop breaks for this tick. (The in-code comment describes a producer-side `depq_pop_min` overflow-drop policy; that is **not actually implemented** — see *Status*.)

### `src/sim/entities.c` — units, incidents, dispatch commit, motion
- `add_unit` appends to a geometrically-growing array and registers the unit in the **AVL** (by id), the **quadtree** (by position), and the **interval tree** (by shift window).
- `sim_init_stations_units` creates exactly 3 stations (one per service), each with 3–5 units, inserts station coverage rectangles into the **R-tree**, and seeds the **treap** workload counter.
- `sim_spawn_incident` is the "one write, many indices" fan-out: a new incident record is threaded into the **B+ tree** (incident log), **red-black tree** and **threaded BST** (both keyed by spawn-time, with an `id*1e-9` tiebreaker), the **DEPQ** (by severity), the **persistent list** (event history), and the **segment tree** (rolling per-second bucket).
- `sim_try_dispatch` is the real dispatch heuristic: quadtree nearest → AVL lookup → Dijkstra per candidate (checks up to 8) → pick min cost; on commit it deletes the incident from the RB/threaded pending indices, inserts into the **splay tree** (recent-units cache), pushes onto the **skip list** ETA board, and bumps the **treap** station-load counter.
- `advance_unit` is the per-unit FSM: ENROUTE→ONSCENE (with a severity-scaled on-scene hold), resolve incident on timer expiry (records response time, deletes from RB/threaded, returns unit to station), or step along the precomputed path at 64 units/sec.

### `src/sim/log.c` — radio log + lazy indices
A fixed 64 KiB rolling FIFO buffer (oldest bytes shifted out via `memmove` on overflow). `sim_log_append` rebuilds the suffix array every 32 writes and the Huffman code every 256 writes, amortizing the cost rather than rebuilding on each line.

### `src/sim/metrics.c` — segment tree + bitvector
Each second-crossing zeroes the next segment-tree bucket (so a range query sums only the live 60-second window), and every tick it **rebuilds and re-`build`s** the succinct bitvector marking which nodes currently host an unresolved incident.

### `src/tui/` — terminal frontend
`main.c` runs a fixed 30-FPS loop (`QueryPerformanceCounter` timing, `Sleep` to cap the frame). `render.c` draws an ANSI, double-buffered city map plus a HUD whose right panel ("DS in action") shows `road_components`, `PQ ops`, `Huffman` ratio, `SA sfx` (suffix count), Huffman-compressed log bytes, and per-frame deltas — explicitly surfacing the DS activity. A `?` help screen lists the DS→feature mapping. The TUI includes only `dispatch/sim.h`; it never reaches into sim internals.

### `web/` — Flask + ctypes frontend
`dispatch_py/__init__.py` is a careful ctypes wrapper: it mirrors every view struct field-for-field, declares `argtypes`/`restype` for every export, lazily loads the DLL (so the module imports even if the DLL isn't built), and exposes an idiomatic `Sim` class returning `NamedTuple` views. Notably, it **deliberately does not free** the malloc'd strings returned by autocomplete/fuzzy, to avoid cross-CRT heap corruption (documented in a comment, `dll._libc_free = None`). `server.py`'s `SimRunner` owns one `Sim` behind a lock and a 30 Hz daemon ticker thread; routes are `/` (static), `/state` (JSON snapshot), `/tick`, `/control` (pause/spawn-rate/force-spawn), `/search` (prefix or fuzzy), and `/log/count` (suffix-array pattern count).

---

## Data structures inventory

The library implements **30 data-structure source files** under `src/ds/**`. The table below maps each to its file, what it powers in the sim, and whether its **query/read side is actually invoked by the running simulation** (verified by grepping `src/sim`) versus only built/maintained as an index and exercised by unit tests. This distinction is the heart of the project.

| Data structure | File | Powers in the sim | Live in sim loop? | Complexity (key op) |
|---|---|---|---|---|
| **Fibonacci heap** | `heaps/fib.c` | Dijkstra priority queue (routing) | **Yes** — push/pop/decrease-key per route | O(1) am. insert & decrease-key; O(log n) am. extract-min |
| **Double-ended PQ (DEPQ)** | `heaps/depq.c` | Severity-ordered pending-incident queue | **Yes** — `depq_pop_max` each tick | O(log n) push/pop-min/pop-max (dual-heap) |
| **DSU / union-find** | `misc/dsu.c` | Connected road components (HUD metric) | **Yes** — at build; count is a metric | ~O(α(n)) (union-by-rank + path compression) |
| **Quadtree** | `spatial/quad.c` | Nearest idle unit to an incident | **Yes** — `sp_quad_nearest`, rebuilt per tick | ~O(log n) avg nearest; lazy 4-way split |
| **Segment tree** | `spatial/seg.c` | Rolling 60-second incident counts | **Yes** — point update + range query | O(log n) update/range-sum |
| **AVL tree** | `trees/avl.c` | Unit roster by id (dispatch lookup) | **Yes** — `tree_avl_get` in dispatch | O(log n) get/insert/delete |
| **Trie (ASCII)** | `strings/trie.c` | Street-name autocomplete | **Yes** — via `sim_autocomplete` | O(L) insert/lookup; O(matches) prefix walk |
| **BK-tree (fuzzy)** | `strings/fuzzy.c` | Typo-tolerant place search | **Yes** — via `sim_fuzzy` | Levenshtein + triangle-inequality pruning |
| **Suffix array** | `strings/sa.c` | Radio-log substring search/count | **Yes** — via `sim_log_count` | O(n log²n) build (prefix doubling) + Kasai LCP; O(m log n) query |
| **Huffman coder** | `trees/huffman.c` | Live log compression ratio (HUD) | **Yes** — rebuilt over log; ratio is a metric | O(n + k log k) build |
| **Succinct bitvector** | `misc/bv.c` | Per-node `has_incident` flag | Built/rebuilt each tick; rank1/select1 **not queried** in sim | O(1) rank, O(log) select (two-level index) |
| **Red-black tree** | `trees/rb.c` | Pending incidents by spawn time | Maintained (insert/delete); never read in sim | O(log n) |
| **Threaded BST** | `trees/threaded.c` | Mirror of pending (stack-free replay) | Maintained; `inorder` never called in sim | O(log n) ins/del; O(n) threaded walk |
| **Splay tree** | `trees/splay.c` | Recently-dispatched unit cache | Inserted into; never read in sim | O(log n) amortized |
| **B+ tree** | `trees/bplus.c` | Incident log / range scans | Inserted into; `get`/`range` never called in sim | O(log n); leaf-linked range scan |
| **Skip list** | `randomized/skip.c` | ETA leaderboard | Inserted into; `rnd_skip_top` never read in sim | O(log n) expected |
| **Treap** | `randomized/treap.c` | Per-station workload ranking | get/insert/delete on commit; never surfaced to UI | O(log n) expected |
| **Compressed radix trie** | `strings/crtrie.c` | Dictionary demo (alt. autocomplete) | Built at startup; `contains` never called in sim | O(L) |
| **DAWG** | `strings/dawg.c` | Minimal automaton for street dictionary | Built+finished at startup; `contains` never called in sim | O(L) lookup; minimal-automaton build |
| **KD-tree** | `spatial/kd.c` | Generic 2D nearest-neighbor | Built once; `sp_kd_nearest` never called in sim | O(log n) avg nearest |
| **R-tree** | `spatial/rtree.c` | Station coverage rectangles | Inserted into; `sp_rtree_search` never called in sim | O(log n) avg search |
| **Interval tree** | `spatial/itree.c` | Unit shift-overlap (stabbing) | Inserted into; `sp_itree_stab` never called in sim | O(log n + k) stabbing |
| **Range tree** | `spatial/range.c` | Orthogonal range reporting | Built once; `sp_range_search` never called in sim | O(log n + k) report |
| **BSP tree** | `spatial/bsp.c` | Static partition of station points | Built once; `sp_bsp_region` never called in sim | O(depth) point location |
| **Persistent list** | `misc/plist.c` | Immutable event history | Pushed to (`misc_plist_push`); `length`/`tail` not read in sim | O(1) push (cons) |
| **Binomial heap** | `heaps/binom.c` | Meldable-PQ demo | Warm-up push/pop only | O(log n) push/pop; O(log n) merge |
| **Leftist heap** | `heaps/leftist.c` | Meldable-PQ demo | Warm-up push/pop only | O(log n) merge-based |
| **Skew heap** | `heaps/skew.c` | Meldable-PQ demo | Warm-up push/pop only | O(log n) amortized merge |
| **Pairing heap** | `heaps/pairing.c` | Meldable-PQ demo | Warm-up push/pop only | O(1) push; O(log n) am. extract |
| **PRNG (splitmix64)** | `randomized/rng.c` | Reproducible level/priority choice | **Yes** — used by skip/treap | O(1) |

> The query-side checks above were verified by grepping `src/sim/**` for each DS's read functions: only `tree_avl_get`, `sp_quad_nearest`, `sp_seg_query`/`sp_seg_update`, `depq_pop_max`, the Fib-heap ops (inside routing.c), DSU ops, the bitvector rebuild, and the string/log search APIs appear. Every other DS is constructed and kept in sync but its read/query API is invoked only by `tests/`.

---

## Key algorithms / implementation details

- **Dijkstra with O(1) amortized decrease-key.** `routing.c` deliberately pairs the Fibonacci heap with a `handles[]` map so an improved tentative distance triggers `heap_fib_decrease_key` rather than a duplicate push, with a stale-entry guard (`e.key > dist[u]`) as a backstop.
- **CSR road graph.** Two-pass degree-count → prefix-sum → scatter (`city.c`), with a parallel `adj_blocked[]` flag so blocked roads are skipped during relaxation without rebuilding the graph.
- **Connectivity via DSU.** Only unblocked roads are unioned; `road_components` is a live HUD number, and the largest component defines the drivable city (`in_main_cc`).
- **Fibonacci heap internals** (`fib.c`): circular doubly-linked root list, lazy O(1) insert, consolidation by degree on extract-min, cut + cascading-cut (via `mark`) on decrease-key — a complete, correct implementation, not a stub.
- **Suffix array** (`sa.c`): prefix-doubling construction (O(n log²n) using `qsort` on (rank, rank) pairs) plus Kasai's O(n) LCP, with substring presence/count answered by two binary searches (lower/upper bound) for an O(m log n) query.
- **BK-tree** (`fuzzy.c`): edges labeled by Levenshtein distance; search prunes children to the `[d−k, d+k]` band using the triangle inequality. Levenshtein itself uses the rolling two-row DP.
- **Succinct bitvector** (`bv.c`): 512-bit blocks, 4096-bit superblocks, `super_rank[]` + `block_rank[]` two-level index, portable SWAR popcount, O(1) rank1 and binary-search select1.
- **Rolling radio log + amortized rebuilds** (`log.c`): bounded 64 KiB FIFO, SA rebuilt every 32 writes, Huffman every 256.
- **Deterministic RNG.** A per-`sim` splitmix64 stream drives all randomness, so a given `(rows, cols, seed)` reproduces exactly; the randomized DS module has its own `rnd_seed` splitmix64 for skip-list levels / treap priorities.

---

## Tech stack & build system

- **Language/standard:** C11 (`-std=c11`), warnings cranked up (`-Wall -Wextra -Wpedantic -Wshadow`), `-O2 -g`, libm only.
- **Compiler target:** MSYS2 / MinGW GCC on Windows (the TUI uses `<windows.h>`, `QueryPerformanceCounter`, `Sleep`; `build.sh` hardcodes `C:/Tools/bin/gcc.cmd` and `C:/msys64/...`).
- **Two equivalent build paths:**
  - `Makefile` — globs `src/ds/**`, `src/sim/*`, `src/tui/*`; builds `libdispatch.a` + `dispatch.exe`, plus `shared` (DLL with import lib) and `tests`.
  - `build.sh` — portable bash fallback with the same targets plus a `check` target that builds and runs all tests.
- **Web stack:** Python 3 + Flask, driving the DLL through pure `ctypes` (no C extension, no subprocess). Frontend is dependency-free HTML/CSS/JS polling `/state`.
- **Artifacts in the tree:** a committed `build/` directory with `.o`/`.exe`/`.a`/`.dll` outputs and a stray `a.exe` at the root (build leftovers; `.gitignore` exists but these are present).

---

## Build / run / test (actual commands)

From the repo root (`C:\Users\ameya\Documents\ADScp`), using bash:

```bash
bash build.sh            # build libdispatch.a + dispatch.exe (TUI)
./build/dispatch.exe     # run the TUI (press '?' for the concept tour)
bash build.sh shared     # build build/libdispatch.dll for FFI / web
bash build.sh tests      # build the 7 test_*.exe binaries
bash build.sh check      # build tests AND run them all
bash build.sh run        # build + launch the TUI
bash build.sh clean
```

Or with GNU make:

```bash
make            # libdispatch.a + dispatch.exe
make shared     # libdispatch.dll (+ import lib)
make tests      # test binaries
make run        # build then run TUI
make clean
```

Web UI:

```bash
make shared                 # produce build/libdispatch.dll
pip install flask
python web/server.py        # serves http://127.0.0.1:5000
# env overrides: SIM_ROWS, SIM_COLS, SIM_SEED; flags: --host --port --debug
# convenience: web/run.sh / web/run.bat build the DLL if missing, then start
```

---

## Tests

Seven standalone test binaries under `tests/`, one per DS module plus an integration test:

- `test_heaps.c`, `test_trees.c`, `test_strings.c`, `test_randomized.c`, `test_spatial.c`, `test_misc.c` — unit-test each DS family directly against the public DS headers, using `assert` and printing `PASS`/exit 0 on success.
- `test_sim.c` — an integration test: creates an 8×8 sim, force-spawns incidents, runs ~600 ticks, then asserts on metrics (incidents/dispatch calls/components/Huffman ratio in [0,1]), verifies autocomplete (`"Abbey"` and a fully-qualified `"Abbey & Harbor"` → exactly 1), runs fuzzy search (`"Maplle"`), validates all snapshot exporters, node-by-street lookup, and log tail/count.
- `web/test_wrapper.py` — a self-test for the ctypes wrapper.

There is **no CI configuration** in the repo and no test runner beyond `build.sh check`. Tests are assertion-based, not a framework. These tests are the *only* place several of the "DS zoo" structures (B+, splay, threaded, kd/R/range/BSP/interval trees, DAWG, crtrie, skip list, treap, plist) have their query paths exercised — the running sim only maintains them as indices.

---

## Status, completeness & notable gaps

**Working and genuinely real:** The simulation runs end to end (TUI and web). The Fibonacci heap, suffix array, BK-tree, DSU, segment tree, succinct bitvector, AVL tree, trie, and Huffman coder are full, correct implementations and are actually used to compute live behavior or HUD metrics. The opaque-handle ABI, the ctypes bridge, and the snapshot/marshalling layer are solid and well-commented.

**Notable gaps / caveats (code-derived):**
- **Many DS are "exhibited," not exercised.** RB tree, threaded BST, splay tree, B+ tree, skip list, treap, compressed radix trie, DAWG, kd-tree, R-tree, range tree, BSP tree, interval tree, and the persistent list are all instantiated and kept in sync, but their *query* APIs are never called inside `src/sim`. The skip-list ETA leaderboard and treap station-load counter, in particular, are written every dispatch but never surfaced to any UI. The four meldable heaps (binomial/leftist/skew/pairing) are only touched by a one-element warm-up in `sim_create`.
- **DEPQ overflow policy not implemented.** `dispatcher.c` comments describe dropping the lowest-severity incident via `depq_pop_min` on overload; the producer (`sim_spawn_incident`) never does this — the pending queue is unbounded.
- **`huffman_encode`/`huffman_decode` and the suffix-array LCP** are implemented but unused by the sim (the sim only uses Huffman's `ratio` and the SA's `count`). The SA source comment even notes "LCP via Kasai (not used externally yet)."
- **`sa_suffix_count` is approximate.** `sim_metrics` reports it as `log.len` when an SA exists, not a recomputed count.
- **Windows-only.** The TUI and `build.sh` are MSYS2/MinGW-specific; `default_dll_path()` in the wrapper does name `.so`/`.dylib`, but the rest of the toolchain assumes Windows.
- **Committed build artifacts** (`build/`, `a.exe`) and **untracked report files** (`Group7.*`, `paper.tex`, `make_ppt.py`) sit in the working tree. No `LICENSE` file.

---

## README vs. code

| README claim | Reality in code |
|---|---|
| Project name is "GridWatch" | The README title is "GridWatch — Emergency Dispatch Simulator," but the code, headers, banners, and docs almost universally call it the "Emergency Dispatch Simulator" / `dispatch`. "GridWatch" appears essentially only in the README title. |
| HUD table: Skip list powers an "ETA leaderboard," Treap a "per-station workload ranking" | Both are written on every dispatch but **never read or displayed**. There is no leaderboard surfaced in TUI or web. The mapping is aspirational. |
| Compressed Trie, DAWG, kd-tree, R-tree, range tree, BSP, interval tree each "used in simulation for …" | All are built/maintained, but their query functions are **not invoked by the running sim** — only by `tests/`. The README's "Used in simulation for" column overstates their runtime role. |
| Binomial / leftist / skew / pairing heaps: "Meldable PQ demo" | Accurate — they are demos. In the sim they only receive a single warm-up push/pop. The DEPQ is the only heap (besides the Fib heap in Dijkstra) doing real work. |
| Syllabus map marks Octree, Big Table, Concurrent DS, Cache-Oblivious DS as "—" | Honest: the README explicitly notes these are not implemented (2D world uses a quadtree; persistence shown via log+plist; sim is single-threaded; B+ noted only as cache-friendly). Code agrees. |
| "PQ ops … stops climbing when paused; resume and watch it spike" | Correct: `pq_operations` only increments inside `sim_dispatch_pending` → Dijkstra, which is gated by `sim_tick`, which is a no-op when paused. |
| "the same `libdispatch.dll` can be bound from Python (`ctypes`/`cffi`)" | True via ctypes; `web/dispatch_py` is a complete ctypes wrapper. No cffi binding exists, and a "future WASM target" is mentioned but not present. |
| Public API: `#include "dispatch/dispatch.h"` umbrella | Exists and includes every module header as described. |
| Three stations of types ambulance/fire/police | Matches `sim_init_stations_units` exactly (always 3 stations, 3–5 units each). |
| Tests print `PASS` and exit 0 | Matches; `test_sim.c` and the per-module tests use `assert` + `printf("PASS")`. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\ADScp. Source of truth: https://github.com/AmeyaBorkar/GridWatch*
