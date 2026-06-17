# SyncUp

> A C-based concurrent directory-mirror / file-backup tool that doubles as a complete Operating Systems course project. The real engine forks one process per top-level subdirectory, each running a pthread worker pool that copies files from a priority queue (largest-file-first), with a custom slab allocator for I/O buffers, mtime/size-based incremental skipping, structured logging, and a resource-allocation-graph deadlock tracker wired into the copy path. Alongside the working backup engine sits a large suite of standalone `--demo` modules and a Python/Tkinter GUI that visualize every classic OS algorithm (CPU scheduling, paging, disk scheduling, Banker's, file-system simulation, classical synchronization).

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/SyncUp-Concurrent-File-Backup-Sync-System |
| **Visibility** | Public |
| **Category** | systems |
| **Primary language(s)** | C (engine + demos), Python/Tkinter (GUI) |
| **Local path** | `C:\Users\ameya\Documents\OperatingSystemCourseProject` |
| **Default branch** | `main` |
| **Lines of code (computed)** | C: 5,453 (4,820 in `src/*.c` + 633 in `include/*.h`); Python: 1,992 (`gui/`, excluding `__pycache__`). Total ≈ 7,445 hand-written LOC. |
| **Source files (computed)** | 15 `.c` files, 14 `.h` files, 15 `.py` files (3 are empty `__init__.py`) |
| **Key components** | `process_manager` (fork/IPC), `thread_pool` (priority queue), `file_io` (copy/scan/log), `memory_pool` (slab allocator), `deadlock_detector` (RAG), `syscall_wrappers`; plus 7 demo modules and a 7-tab Tkinter simulator |
| **License** | None (no `LICENSE` file; README has an informal "free to use/reference for course studies" note) |
| **Last commit** | `e698816` — "Fix Windows console UTF-8 character rendering" (2026-03-24). Working tree is **dirty**: GUI dir, demo data, DLL, `.pyre_configuration`, `.vscode/`, and modified `README.md`/`main.c`/`syncup.exe` are all uncommitted. |

---

## What it actually is

SyncUp is two things bolted into one repo:

1. **A genuinely functional concurrent file-backup CLI** (`syncup.exe` / `syncup`). Given a source and destination directory it mirrors the tree: scans recursively, skips files that are already up to date, and copies the rest in parallel. The concurrency is real — POSIX `fork()` for per-subdirectory processes, pthread worker pools per process, condition-variable-driven work queues, a custom memory allocator for copy buffers, and pipe-based IPC to collect per-child stats. This is the part that maps the "OS concepts" onto an actual application instead of a toy printout.

2. **An OS-course demonstration kit.** Roughly half the C source (7 of the 15 `.c` files) are standalone `--demo` modules that print educational walkthroughs of textbook algorithms (Gantt charts, page-replacement tables, disk seek distances, Banker's safety sequences, inode/bitmap simulations, OS-type comparisons). These are invoked via `syncup --demo <1-6|all>` and are **not** used by the backup engine. The `synchronization.c` demo is the exception — it spins up real pthreads for producer-consumer, readers-writers, and dining philosophers.

3. **A Python/Tkinter GUI** (`gui/`, uncommitted) that re-implements the demo algorithms in pure Python for interactive visualization, plus one tab that shells out to `syncup.exe` and streams its output.

The README's framing ("Advanced Operating Systems Course Project ... maps theoretical OS components into functioning application models") is accurate. The backup engine is the load-bearing, non-trivial code; the demos are educational scaffolding.

---

## Architecture & how it's structured

```
OperatingSystemCourseProject/
├── Makefile                  # gcc -std=c11 -pthread, static link, obj/ build dir
├── README.md                 # (modified, uncommitted)
├── compile_errors.txt        # STALE: log of a failed naive `gcc src\*.c` Windows build
├── syncup.exe                # committed PE32+ x86-64 console binary (~512 KB, static)
├── libwinpthread-1.dll       # winpthreads runtime (present despite -static in Makefile)
├── .pyre_configuration       # Pyre config scoped to gui/ (uncommitted)
├── .vscode/settings.json     # Pylance config (uncommitted)
├── demo_source/  demo_dest/  # sample data; dest contains real run artifacts + .syncup_*.log
│
├── include/                  # 14 headers — one per module
│   ├── process_manager.h     #   ChildReport struct, MAX_CHILD_PROCESSES=32
│   ├── thread_pool.h         #   Task struct, ThreadPoolStats
│   ├── file_io.h             #   IOStats, LogLevel, copy/scan/log API
│   ├── memory_pool.h         #   slab sizes/counts, PlacementStrategy enum, PoolStats
│   ├── deadlock_detector.h   #   MAX_THREADS=64, MAX_RESOURCES=128, RAG + Banker's API
│   ├── syscall_wrappers.h    #   safe_* POSIX wrappers
│   ├── utils.h               #   checksum, size formatting, timestamps
│   └── (scheduler/virtual_memory/disk_scheduler/file_system_sim/
│        os_concepts/process_states/synchronization).h   # demo headers
│
└── src/                      # 15 implementation files
    ├── main.c                # CLI parse, banner, run_backup() vs run_demo() dispatch
    │
    │  ── REAL BACKUP ENGINE ──
    ├── process_manager.c     # fork() per subdir, SIGCHLD/SIGINT handlers, waitpid, pipes
    ├── thread_pool.c         # N pthreads consuming a max-heap priority queue
    ├── file_io.c             # copy_file_buffered, scan_directory_recursive, logger, IOStats
    ├── memory_pool.c         # slab allocator (Part A) + placement/buddy demos (Part B)
    ├── deadlock_detector.c   # RAG adjacency matrix + DFS + Banker's + recovery demo
    ├── syscall_wrappers.c    # open/read/write/close/stat/mkdir/pipe with EINTR retry
    ├── utils.c               # CRC-32 checksum, format_file_size, timestamps, get_time_ms
    │
    │  ── DEMO-ONLY MODULES (printf-driven; not called by the engine) ──
    ├── os_concepts.c         # Unit I: batch/multiprog/timeshare/parallel/realtime(EDF)
    ├── process_states.c      # Unit II: 2/5/7-state models + PCB table
    ├── synchronization.c     # Unit II: producer-consumer, readers-writers, dining phil (REAL pthreads)
    ├── scheduler.c           # Unit III: FCFS/SJF/SRTF/RR/Priority + Gantt chart
    ├── virtual_memory.c      # Unit V: FIFO/LRU/Optimal/Clock paging, TLB, thrashing
    ├── disk_scheduler.c      # Unit VI: FCFS/SSTF/SCAN/C-SCAN seek distances
    └── file_system_sim.c     # Unit VI: inodes, seq/direct/indexed alloc, free bitmap

gui/                          # Python/Tkinter simulator (uncommitted, separate stack)
├── syncup_gui.py             # main window, dark sidebar, lazy-loaded views
├── launch.bat
├── core/                     # pure-Python algorithm logic (no Tkinter)
│   ├── scheduler.py  memory.py  deadlock.py  disk.py
└── views/                    # one Tkinter panel per tab
    ├── scheduler_view.py memory_view.py deadlock_view.py disk_view.py
    ├── process_view.py fs_view.py backup_view.py (shells out to syncup.exe)
```

### Control / data flow (backup mode)

```
main() → parse_args() → run_backup(cfg)
  ├─ safe_stat(src) validate, create dst tree
  ├─ init_logger(dst/.syncup_main.log), setup_signal_handlers(),
  │  memory_pool_init(), deadlock_detector_init()
  │
  ├─ Phase 1: fork_backup_workers(src, dst, num_threads)
  │     opendir(src); for each top-level entry:
  │       • S_ISDIR → safe_pipe(); fork()
  │            child: child_work() → init logger/pool/threadpool,
  │                   scan_directory_recursive() → submits Tasks,
  │                   thread_pool_wait(), write ChildReport to pipe, _exit
  │            parent: record pid, close write-end
  │       • S_ISREG (root-level files) → copied directly in main proc (no fork)
  │
  ├─ Phase 2: monitor_children(reports)
  │     for each child: read ChildReport from pipe, waitpid(), report status
  │
  ├─ Phase 3: rag_detect_deadlock()  [+ rag_print_graph() if -v]
  │
  └─ print_process_summary + io_print_stats + pool_print_stats + total time
     → deadlock_detector_destroy, memory_pool_destroy, close_logger
```

Inside each worker process the data path is:
`scan_directory_recursive()` → `file_needs_backup()` (skip or) → `thread_pool_submit(src,dst,size)` → `heap_push` onto a max-heap → worker thread `heap_pop()` → RAG request/acquire → `copy_file_buffered()` (pool buffer + `safe_read`/`safe_write` loop) → RAG release → stats update.

---

## Code walkthrough (the important parts)

### `main.c` (293 lines)
Entry point. `Config` struct holds `src_dir`, `dst_dir`, `num_threads` (default 4, clamped to `MAX_THREADS`=64), `verbose`, and `demo_mode` (0 = backup; 1–6 = unit; 7 = all). `parse_args` scans for `--demo`/`-h` first, then treats `argv[1]`/`argv[2]` as src/dst and parses `-t`/`-v`. On `_WIN32` it calls `SetConsoleOutputCP(CP_UTF8)` so the box-drawing banner renders (that's exactly what the last commit fixed). `run_backup` orchestrates the three phases above; `run_demo` dispatches to the demo modules. Note `run_demo` calls `demo_synchronization` and `demo_process_states` for unit 2 but **does not** call any `demo_process_management` (the real process code only runs in backup mode).

### `process_manager.c` (333 lines) — multiprocessing + IPC
The heart of "process management." Module-level state: `child_pids[32]`, `pipe_fds[32][2]`, `num_children`, and two `volatile sig_atomic_t` flags. 

- **Signal handling**: `setup_signal_handlers` installs `sigaction` for `SIGCHLD` (with `SA_RESTART | SA_NOCLDSTOP`) and `SIGINT`. The `SIGINT` handler forwards the signal to every child via `kill()` for graceful shutdown. On `_WIN32` it falls back to plain `signal(SIGINT, …)` and prints that only SIGINT is installed.
- **`fork_backup_workers`**: `opendir` the source; for each top-level **directory** it creates a pipe and `fork()`s a child; for each top-level **regular file** it sets `has_root_files` and, after the fork loop, copies those root files synchronously in the parent (no dedicated process). Children close the read-end and call `child_work`; the parent closes the write-end and records the pid.
- **`child_work`**: per-child it opens its own log (`dst/.syncup_<pid>.log`), re-inits memory pool / deadlock detector / thread pool, runs `scan_directory_recursive`, `thread_pool_wait`s, gathers `IOStats`, builds a `ChildReport` (pid, files copied/skipped, bytes, elapsed, directory) and `safe_write`s it down the pipe before tearing everything down.
- **`monitor_children`**: reads each `ChildReport` from its pipe, then `waitpid`s and prints `WIFEXITED`/`WIFSIGNALED` status.
- **Windows reality**: the entire fork path is `#ifdef _WIN32`-guarded — on Windows `child_work` is invoked **synchronously in the main process** ("Windows fallback: running worker synchronously"), `child_pids[i]` is set to a fake sequential id, and `monitor_children` just prints "completed" with no `waitpid`. The committed run artifacts confirm this: all three subdirectory logs in `demo_dest/` share the same PID `9772`.

### `thread_pool.c` (293 lines) — priority work queue
A fixed-capacity (`MAX_TASKS`=4096) **binary max-heap** keyed on `Task.priority`, guarded by one mutex (`queue_lock`) and two condition variables (`queue_cond` for work-available, `done_cond` for completion). `thread_pool_submit` computes priority from file size (3 = >1 MB, 2 = >64 KB, 1 = >4 KB, 0 = small) and `heap_push`es. Workers loop: wait on `queue_cond` until a task or `shutdown_flag`, `heap_pop`, run the copy, then `tp_stats.tasks_completed++` and broadcast `done_cond` when `completed == submitted && queue empty`. `thread_pool_wait` blocks on `done_cond`. Each worker also threads the RAG calls (`resource_id_for_path` → `rag_request_resource` → on deadlock, `heap_push` re-queue and `continue`; else `rag_acquire_resource` … `rag_release_resource`). Destroy sets `shutdown_flag`, broadcasts, and `pthread_join`s all workers.

### `file_io.c` (297 lines) — copy / scan / incremental / logging
- **`copy_file_buffered`**: picks a buffer size by file size (64 KB for >256 KB, 16 KB for >8 KB, else 4 KB), `pool_alloc`s it from the slab allocator, opens src `O_RDONLY` and dst `O_WRONLY|O_CREAT|O_TRUNC` (0644) via the safe wrappers, copies in a `safe_read`/`safe_write` loop, then `pool_free`s and updates `io_stats` (under `stats_lock`).
- **`file_needs_backup`** (the "diff/sync" logic): copy if dst doesn't exist, **or** `src.st_mtime > dst.st_mtime`, **or** sizes differ. Otherwise skip. This is mtime+size incremental, *not* content hashing (the CRC-32 in `utils.c` exists but is never called by the engine).
- **`create_directory_tree`**: `mkdir -p` by walking the path and `safe_mkdir`ing each prefix.
- **`scan_directory_recursive`**: `opendir`/`readdir`, recurse into subdirs (mirroring them), and for regular files increment `files_scanned`, then either `thread_pool_submit` or count as skipped.
- **Logger**: append-mode `FILE*` guarded by `log_lock`; every message timestamped with level; WARN/ERROR also echoed to stderr. `fflush` after each line.

### `memory_pool.c` (568 lines) — slab allocator + placement demos
- **Part A (used by the engine)**: three `Slab`s of fixed block sizes — 4 KB×64, 16 KB×32, 64 KB×16 — each a `malloc`'d arena threaded into a singly-linked free-list of `FreeNode`s. `pool_alloc` picks the smallest slab whose `block_size >= size`, pops a node, and tracks `internal_frag += block_size - size`. If no slab fits (or it's empty) it falls back to plain `malloc` (counted in `fallback_mallocs`). `pool_free` checks pointer bounds to decide whether to return a node to its slab or `free` a fallback. All under `pool_lock`; tracks peak/current usage.
- **Part B (demo-only)**: First/Best/Next/Worst-fit dynamic partitioning with split + coalesce, a buddy-system allocator (power-of-2 split/merge), and an internal-vs-external fragmentation walkthrough.

### `deadlock_detector.c` (442 lines) — RAG + Banker's
- **RAG** (`rag` is a `(MAX_THREADS+MAX_RESOURCES)²` adjacency matrix of `EdgeType`): `rag_request_resource` adds a request edge and does a quick two-step cycle check against the current holder; `rag_acquire_resource` flips it to an assignment edge and records the holder; `rag_release_resource` clears it. `resource_id_for_path` interns file paths into resource IDs. `rag_detect_deadlock` runs a full **DFS cycle detection** over the whole graph and also flags threads waiting longer than `DEADLOCK_TIMEOUT_MS`=3000. All guarded by `rag_lock`.
- **`bankers_check_safe`**: textbook safety algorithm computing a safe sequence from available/alloc/max matrices.
- The rest (`demo_deadlock_*`) are educational printouts (prevention strategies, recovery via lowest-priority victim selection).

### `syscall_wrappers.c` (138 lines) & `utils.c` (101 lines)
`safe_open/read/write/close/stat/mkdir/pipe` — thin POSIX wrappers with `EINTR` retry loops; `safe_write` also loops over partial writes; `safe_stat` suppresses the expected `ENOENT`; `safe_mkdir` tolerates `EEXIST`. `_WIN32` branches use `mkdir(path)` (1-arg) and `_pipe(fd, 4096, _O_BINARY)`. `utils.c` has a real CRC-32 (`0xEDB88320` polynomial — defined but unused by the engine), `format_file_size`, `get_timestamp_string` (`localtime_r`), `print_progress_bar`, and `get_time_ms` (CLOCK_MONOTONIC).

### Demo modules (`os_concepts`, `process_states`, `scheduler`, `virtual_memory`, `disk_scheduler`, `file_system_sim`, `synchronization`)
All seven are reached only via `--demo`. Six are pure `printf` simulations over hardcoded data:
- `scheduler.c`: FCFS, SJF (NP), SRTF, Round Robin, Priority (NP+P) with a `SchedProcess` struct and TAT/WT/RT/response-time metrics + Gantt chart.
- `virtual_memory.c`: page-table + 4-entry TLB, FIFO/LRU/Optimal/Clock page replacement over a fixed reference string, thrashing detection (rolling 10-access/7-fault window).
- `disk_scheduler.c`: FCFS/SSTF/SCAN/C-SCAN total-head-movement comparison over an 8-request set.
- `file_system_sim.c`: inode table, sequential/direct/indexed allocation, bitmap free-space management, parent-pointer directory hierarchy.
- `os_concepts.c`: batch / multiprogramming / time-sharing / parallel (2 CPUs) / real-time (EDF).
- `process_states.c`: 2/5/7-state process models with PCB table and transition history.

The one genuinely concurrent demo is **`synchronization.c`**: it implements a custom counting semaphore (mutex+condvar), a monitor abstraction, and runs producer-consumer (bounded buffer size 5), readers-writers, and dining philosophers (deadlock-avoided by lower-fork-first ordering) with **real pthreads**.

### GUI (`gui/`, Python/Tkinter, uncommitted)
`syncup_gui.py` is a dark Catppuccin-Mocha-themed window with a sidebar of 7 tabs and lazy-loaded view classes. `core/*.py` are independent pure-Python reimplementations of the algorithms (e.g. `core/disk.py` implements FCFS/SSTF/SCAN/C-SCAN **and LOOK** — LOOK exists only in the GUI, not in the C `disk_scheduler.c`). `views/backup_view.py` is the only bridge to the C engine: it locates `syncup.exe`, builds the command line (`exe src dst -t <n> [-v]`), runs it via `subprocess.Popen` in a thread, strips ANSI codes, and streams classified output into a console widget. It pre-fills source/dest with `demo_source`/`demo_dest`.

---

## Key algorithms / implementation details (verified)

- **Concurrency model: process-per-subdirectory × thread-pool-per-process.** Real `fork()` on POSIX (one child per top-level subdir, capped at 32), each child running `num_threads` pthreads. On Windows the fork layer is stubbed to synchronous in-process execution — so on the only platform the committed binary targets, the "multiprocessing" degenerates to a single process with thread pools run sequentially per subdirectory.
- **Task scheduling: max-heap priority queue keyed on file size.** Larger files get higher priority and are scheduled first (header comment says "min-heap" but the code is unambiguously a **max-heap** — `> parent` sift-up, `> largest` sift-down). Mutex + two condvars; completion detected by `tasks_completed == tasks_submitted && queue empty`.
- **Sync/diff algorithm: mtime-or-size incremental.** `file_needs_backup` copies when the dest is missing, the source mtime is newer, or sizes differ. No checksum comparison (CRC-32 exists but is unused), no deletion of extra dest files, no two-way sync — it's a one-way mirror/backup, not a bidirectional sync.
- **IPC: one anonymous `pipe()` per child**, used to send a single fixed-size `ChildReport` struct from child to parent. Parent reads it in `monitor_children` then `waitpid`s.
- **Deadlock handling in the live path:** every file copy interns its path as a "resource" and requests/acquires/releases it in the RAG. If a request would cause a cycle (quick holder check), the worker backs off and re-queues the task. A final full DFS cycle check runs in Phase 3. In practice each path maps to a unique resource, so genuine deadlock is essentially impossible here — the RAG is wired in primarily to *demonstrate* the mechanism on the real workload.
- **Custom allocator: 3-class slab** (4/16/64 KB) with free-lists and `malloc` fallback, used for copy buffers; tracks internal fragmentation, peak/current usage, and fallback counts.
- **EINTR-safe syscalls + partial-write handling** throughout the I/O layer.

---

## Tech stack & build system

- **Language/standard**: C11 (`-std=c11`), POSIX (`_POSIX_C_SOURCE=200809L`), pthreads.
- **Toolchain**: `gcc` (MinGW/MSYS2 on Windows). The `Makefile` uses `-Wall -Wextra -pthread`, compiles `src/*.c` to `obj/*.o`, links with `-pthread -static`, output `syncup`. Targets: `all`, `clean`, `test` (generates a sample tree incl. a 256 KB random file and runs a `-t 4 -v` backup), `demo` (`--demo all`), `help`.
- **GUI**: Python 3.8+ with Tkinter (stdlib). Static typing config via `.pyre_configuration`/`pyrightconfig.json`/Pylance, all scoped to `gui/` and mostly with error-checking disabled.
- **Committed binary**: `syncup.exe` is a static-ish PE32+ x86-64 console build; `libwinpthread-1.dll` ships alongside it (so despite `-static` in the Makefile, the committed binary still expects the winpthreads DLL).

---

## Build / run / test (actual commands)

```bash
# Build (Linux/MSYS2 with make):
make                       # → ./syncup (or syncup.exe)

# Build (Windows, naive one-liner from README):
gcc src\*.c -Iinclude -pthread -o syncup.exe
#   NOTE: this exact one-liner is what compile_errors.txt records FAILING earlier
#   (sys/wait.h, mkdir arity, SCHED_RR collision). The current source has added
#   #ifdef _WIN32 guards, so it builds on MinGW now — but a clean MinGW gcc is required.

# Run a backup:
./syncup <source_dir> <dest_dir> [-t <threads>] [-v]
./syncup demo_source demo_dest -t 4 -v
./syncup C:\Source C:\Dest -t 8 -v

# Run OS-concept demos:
./syncup --demo 3          # CPU scheduling
./syncup --demo all        # everything
make demo                  # == ./syncup --demo all

# Convenience test (creates test_source/, runs backup, verifies trees):
make test

# GUI:
cd gui && python syncup_gui.py     # or double-click gui/launch.bat
```

`gcc`/`make` were **not on PATH** in the environment used to generate this doc, so a from-scratch build could not be re-verified here — but the committed `syncup.exe` is a valid 512 KB x86-64 binary and `demo_dest/` contains real run artifacts (logs timestamped 2026-03-24), proving the engine has been executed successfully on Windows.

## Tests

There is **no automated test suite** — no unit-test framework, no CI config, no test sources. "Testing" is the `make test` target: it scaffolds a sample directory tree (documents/photos/code plus a 256 KB `/dev/urandom` binary), runs `./syncup test_source test_dest -t 4 -v`, then `find`s both trees and prints them for a manual eyeball diff. The `demo_source`/`demo_dest` pair in the repo is effectively a checked-in manual test fixture.

## Status, completeness & notable gaps

- **Working**: the backup engine compiles and runs (committed binary + run artifacts confirm it). Recursive scan, incremental skip, threaded copy, slab buffers, per-file logging, and the stats summary all function. The demos and GUI are also functional.
- **Windows fork stub**: the headline "multiprocessing via fork()" only truly happens on POSIX. On Windows (the committed target) it runs synchronously in one process — verified by all child logs sharing PID 9772. So the parallelism on the shipped platform is thread-level only.
- **One-way mirror, not "sync"**: despite "Sync System" in the name, there's no bidirectional sync, no deletion propagation, and no conflict handling. Diffing is mtime/size only; the CRC-32 checksum is dead code.
- **No `-h`/usage for an unknown flag, minimal arg validation**; thread count clamped 1..64.
- **Demos vs. engine**: ~half the C source is educational printf modules unrelated to backup; that's by design for a course project but worth knowing when assessing "what the system does."
- **No license file.** README has only an informal usage note.
- **Dirty tree**: the entire `gui/` directory, the demo data, the DLL, and editor/type-checker configs are uncommitted, as is a README rewrite and the cosmetic `main.c` banner tweak. Committed `__pycache__` `.pyc` files are staged (build noise).
- **`compile_errors.txt`** is a stale failure log left in the repo; it does not reflect the current (guarded) source.

## README vs. code

| README claim | Code reality |
|---|---|
| "high-performance, concurrent directory mirror using **multiple processes**" | True on POSIX (`fork()` per subdir). **On Windows the fork path is `#ifdef`-stubbed to synchronous single-process execution** — run logs all share PID 9772. README never mentions this caveat. |
| thread_pool.h comment: "priority queue (**min-heap** keyed on file size)" | It's a **max-heap** (largest file = highest priority, scheduled first). Header comment is wrong; `thread_pool.c`'s own comment correctly says max-heap. |
| Unit IV "Prevention: **Lock ordering via alphabetical filepath sort**" | No alphabetical filepath sort exists. Prevention is only described in a printf demo (`demo_deadlock_prevention`) as ascending resource-ID ordering; the live engine uses RAG request/back-off, not a sorted lock order. |
| Build: `gcc src\*.c -Iinclude -pthread -o syncup.exe` | This is the **exact command `compile_errors.txt` shows failing**. It works only after the `#ifdef _WIN32` guards now present in the source, with a proper MinGW gcc. README presents it as if it just works. |
| Disk scheduling lists "FCFS / SSTF / SCAN / C-SCAN / **LOOK**" (GUI table) | The C `disk_scheduler.c` implements FCFS/SSTF/SCAN/C-SCAN only. **LOOK exists solely in the Python GUI** (`gui/core/disk.py`). |
| "synthesized as an educational proof-of-concept" / "free to use or reference" | Matches: no formal license, course-project framing. The backup engine is more than a proof-of-concept, though — it's a real working tool. |
| README architecture says "14 Component Headers / 15 Implementation Sources" | Confirmed exactly: 14 `.h`, 15 `.c`. |
| GUI section ("ships with a dark-themed animated desktop GUI") | Accurate, but the entire `gui/` tree is **uncommitted** working-tree content, not part of any commit yet. |
| README implies process-state / sync demos under `--demo 2` incl. `process_manager.c` | `--demo 2` runs `demo_process_states` + `demo_synchronization` only. `process_manager.c` has **no demo** — its code runs only in real backup mode. |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\OperatingSystemCourseProject. Source of truth: https://github.com/AmeyaBorkar/SyncUp-Concurrent-File-Backup-Sync-System*
