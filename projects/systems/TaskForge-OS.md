# TaskForge-OS

> A C11 "operating system" (a single-process userland simulation, not a bootable kernel) that exposes a syscall-style API — `sys_fork`, `sys_alloc`, `sys_open`, `sys_lock`, `sys_read/write`, etc. — and runs a banking application entirely through that API. Every deposit/withdraw/transfer creates a PCB, allocates from a free-list memory pool, takes a Banker's-validated resource lock, reads/writes an in-memory file node, touches an LRU/FIFO/Clock page cache, and issues a disk-scheduling request, with the whole stack traced live. The repo also ships a standalone v1 OS-concepts simulator (`src/*.c`) and a Python/Flask port that mirrors the kernel for a browser dashboard.

| | |
|---|---|
| **Repository** | https://github.com/AmeyaBorkar/TaskForge-OS |
| **Visibility** | Public |
| **Category** | systems |
| **Primary language(s)** | C (C11) — kernel, banking app, GUIs; Python 3 + Flask — web port; HTML/CSS/JS — web SPA |
| **Local path** | `C:\Users\ameya\Documents\TaskForge\TaskForge` |
| **Default branch** | `main` |
| **Lines of code (computed)** | C: 11,705 (`.c`) + 782 (`.h`) = **12,487**; Python: 1,340; HTML: 874. Tracked source total ≈ **15,201** non-blank+blank lines across the languages below. |
| **Source files (computed)** | 12 `.c`, 10 `.h`, 2 `.py`, 1 `.html`, plus `Makefile`, `build.bat`, 2 `.md` (29 tracked text/source files total, by `git ls-files`) |
| **Key components** | `kernel/os_kernel.c` (the "OS"), `kernel/os_syscall.h` (syscall API), `app/bank.c` (banking app), `main_v2.c` (boot + menu), `web/app.py` (Flask port), `src/*.c` (v1 simulator), `gui/*.c` (Win32 GUIs) |
| **License** | None found (no `LICENSE` file in the tree) |
| **Last commit** | `03e14c1` — "Update project report (docx)", 2026-04-28 |

---

## What it actually is

TaskForge-OS is an **Operating Systems course project** with three coexisting front-ends over (mostly) the same conceptual kernel:

1. **v2 — Banking on the kernel (the headline project).** `main_v2.c` "boots" a global `Kernel g_kernel` (`kernel_init`), then runs a banking app (`app/bank.c`) whose every operation goes through the syscall layer (`kernel/os_syscall.h`). The kernel is **not** a bootable OS and runs **no scheduler thread** for the bank — it is an in-process library of OS-concept data structures (process table, free-list allocator, page cache, in-RAM file system, Banker's matrices, disk-queue scheduler) protected by pthread mutexes. The "system calls" are `static inline` C wrappers that log and forward to `kernel_*` functions.

2. **v1 — Standalone simulator.** `main.c` + `src/*.c` is an unrelated interactive menu app that demonstrates each syllabus algorithm in isolation (enter your own process set / reference string / disk queue and watch it run). This is where the *real* concurrency (pthreads: Producer-Consumer, Readers-Writers, Dining Philosophers) and the *real* file operations (grep, XOR cipher, RLE compress, sort, checksum, copy, batch rename on actual disk files) live — none of which are in the kernel.

3. **v2 web port.** `web/app.py` is a Flask backend that **re-implements the kernel in Python** (`class Kernel`) plus the bank (`class BankApp`), serving a single-page dark-theme dashboard (`web/templates/index.html`) with Banking / OS Dashboard / OS Concepts / Configuration tabs. The README calls this the recommended way to demo.

There are also two Win32 GUIs (`gui/taskforge_gui.c` for v1, `gui/taskforge_gui_v2.c` for v2) that link the same C kernel.

The project's pedagogical hook (stated in `CONTRIBUTIONS.md`): a single banking transaction exercises all six syllabus units, and you can swap the scheduler / memory strategy / cache policy / disk algorithm at runtime and watch the same operation take a different path.

---

## Architecture & how it's structured

Layered design, identical concept in C (v2) and Python (web):

```
+---------------------------------------------------------------+
|  BANKING APPLICATION         app/bank.c   (web: BankApp)      |
|  create / deposit / withdraw / transfer / balance / statement |
+---------------------------------------------------------------+
|  SYSTEM CALL API             kernel/os_syscall.h              |
|  30 static-inline wrappers: sys_fork, sys_alloc, sys_open,    |
|  sys_lock, sys_read/write, sys_io_request, sys_set_* ...      |
|  (each logs to the kernel ring buffer, then calls kernel_*)   |
+---------------------------------------------------------------+
|  TASKFORGE "OS KERNEL"       kernel/os_kernel.c (1576 LOC)    |
|  S1 boot/shutdown/log   S2 process mgmt + scheduler           |
|  S3 memory (free-list)  S4 page cache   S5 file system        |
|  S6 deadlock (Banker)   S7 I/O (disk scheduling)              |
|  one global  Kernel g_kernel ; 6 pthread_mutex_t locks        |
+---------------------------------------------------------------+
```

Annotated tree (kernel/app/web are the live v2 path; src/include is the separate v1 app):

```
TaskForge/
├── kernel/
│   ├── os_kernel.h        # 328 LOC: all kernel types (PCB, MemBlock, CacheEntry,
│   │                      #   FSNode, FileDescriptor, DeadlockMgr, IOSubsystem,
│   │                      #   Kernel) + constants + API prototypes
│   ├── os_kernel.c        # 1576 LOC: the kernel implementation, 7 sections
│   └── os_syscall.h       # 172 LOC: 30 static-inline sys_* wrappers + extern g_kernel
├── app/
│   ├── bank.h             #  59 LOC: Account, Transaction, BankState
│   └── bank.c             # 788 LOC: banking ops, OS trace printing, bank_menu
├── main_v2.c              # 384 LOC: boot kernel, kernel dashboard, OS config menu, main menu
├── main.c                 # 155 LOC: v1 entry — menu into src/* simulator modules
├── include/               #  v1 module headers (mostly 1-line prototypes, 8-12 LOC each)
├── src/                   #  v1 standalone simulator (NOT used by v2):
│   ├── scheduler.c        # 767  — FCFS/SJF/SRTF/RR/Priority(NP&P) + Gantt + compare
│   ├── deadlock.c         # 715  — Banker's, RAG (DFS cycle), detection, prevention, recovery
│   ├── memory_mgmt.c      #1074  — fixed/dynamic part., buddy, paging, TLB, segmentation,
│   │                      #         page replacement (FIFO/LRU/Optimal/Clock), thrashing
│   ├── process_mgmt.c     # 951  — 2/5/7-state models; REAL pthreads: producer-consumer,
│   │                      #         readers-writers, dining philosophers (sem_t / mutex)
│   ├── io_file_mgmt.c     # 817  — disk sched, buffering schemes, in-mem VFS shell, free-space
│   └── task_ops.c         # 793  — REAL disk file ops: grep, wc, XOR cipher, RLE, sort,
│   │                      #         checksum, copy w/ progress, batch rename
│   └── (common.h          # 167  — shared colors, input helpers, macros, in include/)
├── gui/
│   ├── taskforge_gui.c    #2104  — Win32 GUI for v1 simulator
│   └── taskforge_gui_v2.c #1581  — Win32 GUI for v2 (banking + dashboard + config + trace)
├── web/
│   ├── app.py             # 721  — Flask: Python re-impl of Kernel + BankApp + 13 REST routes
│   └── templates/index.html # 874 — single-page dashboard (4 tabs)
├── Makefile               #  80  — builds v1 only (taskforge) on POSIX gcc + -lpthread
├── build.bat              # 104  — Windows/MSYS2 builder: v2 | v1 | gui | all | clean
├── generate_report.py     # 619  — builds the .docx project report
├── CONTRIBUTIONS.md       # team / concept-ownership doc
└── README.md
```

**Control flow (v2 deposit):** `bank_menu` → `bank_deposit` → `proc_start` (`sys_fork` → `kernel_create_process`; then two `sys_set_state` calls NEW→READY→RUNNING) → `proc_mem_alloc` (`sys_alloc`) → `sys_set_max_need` + `sys_lock` (`kernel_resource_request` runs Banker's safety check) → `sys_cache_access` → `read_account_file` (`sys_open`/`sys_read`/`sys_close`) → balance updated in `BankState` → `write_account_file` (`sys_open`/`sys_write`/`sys_close`) → `disk_io_for_account` (`sys_io_request` + `sys_io_process`) → `sys_unlock` → `proc_cleanup` (`sys_free_all` + state→TERMINATED). Each step prints an `[OS]` trace line and appends to the kernel log ring buffer.

**Data flow:** the canonical balance lives in `BankState.accounts[]` (plain C struct in the app). The kernel's file system stores a serialized `"id|holder|balance"` string per account in an `FSNode.data` byte buffer, written/read through fds, but the app does not actually parse the read-back data into the balance — the file I/O is exercised for the trace, while the authoritative number stays in `BankState`.

---

## Code walkthrough (the important parts)

### `kernel/os_kernel.h` — the data model
Defines fixed-capacity tables: `OS_MAX_PROCESSES 64`, `OS_MAX_RESOURCES 32`, `OS_MAX_FILES 64`, `OS_MAX_OPEN_FDS 128`, `OS_MAX_MEM_BLOCKS 256`, `OS_MEMORY_SIZE 4096` KB, `OS_CACHE_SIZE 16`, `OS_DISK_SIZE 200` tracks, `OS_MAX_LOG 256`. The `PCB` carries `state`, `priority`, `burst_time`/`remaining_time`, scheduling stats, `resources_held[]`/`resources_max[]`, and an (unused-in-bank) `entry`/`arg`/`pthread_t thread`. The `Kernel` struct aggregates every subsystem plus six `pthread_mutex_t` (proc/mem/fs/dl/io/log).

### `kernel/os_syscall.h` — the syscall boundary
30 `static inline` wrappers around `kernel_*`. Each process/memory/fs/lock/disk wrapper calls `kernel_log(&g_kernel, "SYSCALL: ...")` first, then forwards. `sys_read`/`sys_write` additionally bump `g_kernel.io.buf_reads`/`buf_writes` directly (a small layering leak — touching kernel state from the syscall header). `g_kernel` is `extern` here and defined in `os_kernel.c`.

### `kernel/os_kernel.c` — Section by section

- **S1 Boot/shutdown/log.** `kernel_init` zeroes the struct, inits the 6 mutexes, seeds memory as one 4096 KB free block, creates the root `/` FSNode, sets defaults (FCFS, First-Fit, LRU, FCFS-disk, disk head at track 100), marks 16 cache slots empty, and sets `deadlock.available[r] = 10` for `r<5` (0 for the rest) with `prevention_on = 0`. `kernel_log` is a thread-safe **circular ring buffer** (`log_head`, `log_count`, modulo `OS_MAX_LOG`) using `vsnprintf`.

- **S2 Process management.** `kernel_create_process` finds a free PCB slot, assigns `next_pid++`. `kernel_set_state` enforces a **validated 5-state transition table** (NEW→READY→RUNNING→{READY,WAITING,TERMINATED}, WAITING→READY) and computes turnaround/wait on termination. `kernel_kill_process` frees all the process's memory and releases all its resources back to `available[]`. `kernel_schedule_next` implements FCFS (min arrival), SJF (min remaining), Priority (min priority value = highest), and Round Robin (rotating `static int last_slot` + quantum decrement). **Important:** nothing in the v2 bank or `main_v2.c` ever calls `kernel_schedule_next`/`sys_schedule` — the bank drives state directly via `sys_set_state`. The scheduler exists and is configurable but is dormant in the live banking path; it is only reachable through the dashboard's algorithm display. (The web port *does* call `schedule_next` inside `BankApp._boot_proc`.)

- **S3 Memory.** Real **contiguous free-list allocator** with four placement strategies: First/Best/Worst/Next Fit. `kernel_mem_alloc` finds a block, **splits** the remainder by shifting the block array, marks owner, updates the PCB, and — as a side effect — fires a random `kernel_io_request` to simulate a page load. `kernel_mem_free` and `kernel_mem_free_all` **coalesce** adjacent free blocks. `kernel_mem_stats` reports used/free/blocks and **fragmentation = 1 − largest_free/total_free**.

- **S4 Page cache.** `kernel_cache_access` returns hit/miss and on miss selects a victim by **LRU** (min `last_access`), **FIFO** (min `load_time`), or **CLOCK** (second-chance sweep over `ref_bit`, up to `2*size` passes, advancing `clock_hand`). Empty slots are always preferred first. Stats track hits/misses and hit ratio. (Note: on a miss it sets both `last_access = cache_tick++` and then `load_time = cache_tick` *without* a further increment, so the two tick fields are off by one — harmless.)

- **S5 File system.** Flat-array VFS with parent-id links. `kernel_fs_create`/`delete` (delete refuses non-empty dirs and the root), `kernel_fs_find` (by name+parent), `kernel_fs_open` (allocates an fd with mode bits 1=read/2=write), `kernel_fs_read`/`write` (bounded by `OS_IO_BUF_SIZE 4096`, enforce mode permission bits, advance offset; read triggers a `cache_access`, write triggers a random `io_request`), and `kernel_fs_list`.

- **S6 Deadlock.** `kernel_banker_safe` is the classic **Banker's safety algorithm** (Work/Finish vectors over Need/Allocation; inactive PCB slots pre-finished). `kernel_resource_request` optionally enforces **resource ordering** (only when `prevention_on`), checks request≤need and request≤available, **tentatively allocates, runs the safety check, and rolls back if unsafe**. `kernel_deadlock_check` runs the detection variant and collects stuck pids, incrementing `deadlock_count`. `kernel_set_max_need` populates the Need matrix.

- **S7 I/O.** `kernel_io_request` enqueues a track; `kernel_io_process` serves one request by **FCFS, SSTF (min seek), SCAN (nearest in current direction, reverse if none), or C-SCAN (nearest ahead, else jump to lowest)**, accumulating `total_seek` and `total_ops`.

### `app/bank.c` — the banking application
`BankState` holds up to 20 accounts and 100 transactions. `bank_init` creates `/accounts` and `/logs` dirs via syscalls. Helper wrappers (`proc_start`, `proc_mem_alloc`, `proc_cleanup`, `write_account_file`, `read_account_file`, `disk_io_for_account`) make every op narrate the full kernel path. `bank_transfer` is the deadlock showpiece: it **locks the lower account id first** (resource ordering), validates each lock through Banker's, rolls back the first lock if the second is denied, and unlocks in reverse order. `bank_menu` is the 0–9 interactive menu; `bank_os_dashboard` prints live kernel stats and the recent log.

### `main_v2.c` — boot + control panels
Prints a boot sequence, calls `kernel_init` and `bank_init`, then a main menu (Banking / Kernel Dashboard / OS Configuration / About). `print_kernel_dashboard` renders processes, memory, cache, deadlock, FS, and I/O panels. `os_config_menu` lets you change scheduler/memory/cache/disk algorithms at runtime via `sys_set_*` and run a manual deadlock check.

### `web/app.py` — Python parallel kernel
A faithful Python re-implementation: `class Kernel` mirrors the C subsystems (10 resources of 10, 16 cache slots, 4096 KB) and `class BankApp` mirrors the C bank. Notable divergences from C: **`prevention_on = True` by default** (vs C's `0`); `_boot_proc` actually calls `schedule_next`; deposit/withdraw use priority 2 burst 8 (C uses 4/5). Exposes 13 Flask routes including `/api/concepts`, which returns a hard-coded syllabus-unit → file/owner/live-metric mapping for the course showcase.

---

## Key algorithms / implementation details (verified)

| Area | Algorithms in the **C kernel** (`os_kernel.c`) | Algorithms in **v1 simulator** (`src/*.c`) only |
|---|---|---|
| CPU scheduling | FCFS, SJF (non-preemptive, by remaining), Priority (lower=higher), Round Robin (quantum) — **but dormant in the bank path** | FCFS, SJF, **SRTF**, RR, Priority (preemptive + non-preemptive), Gantt charts, compare-all |
| Memory placement | First / Best / Worst / Next Fit, block split + coalesce, fragmentation metric | Fixed/dynamic partitioning, **buddy system** |
| Virtual memory | (none — no paging/TLB/segmentation in kernel) | **paging address translation, TLB (LRU), segmentation, thrashing** |
| Page replacement | LRU, FIFO, Clock (second-chance) — used as the account page cache | FIFO, LRU, **Optimal (Belady)**, Clock + comparison |
| Deadlock | Banker's safety (avoidance), tentative-alloc-then-rollback, detection, optional resource ordering | Banker's, **RAG with DFS cycle detection**, detection, prevention demos, recovery |
| Disk scheduling | FCFS, SSTF, SCAN, C-SCAN with seek accounting | FCFS, SSTF, SCAN, C-SCAN + buffering schemes + free-space (bitmap/linked) |
| Syscall mechanism | 30 `static inline` wrappers, log-then-forward to one global `g_kernel` | n/a |
| Concurrency | mutex-protected kernel; **no worker threads spawned by the bank** | **real pthreads + sem_t**: producer-consumer, readers-writers, dining philosophers |

**Deadlock detail worth noting:** the bank registers exactly one unit of need per account and acquires single locks, so the Banker's check effectively always reports SAFE in normal use — the value is demonstrating the *mechanism* (tentative allocate → safety check → rollback), not provoking real contention. There is no real concurrent contention in v2 because operations run sequentially on the menu thread.

---

## Tech stack & build system

- **C11**, compiled with GCC, `-Wall -Wextra -std=c11`, linked with `-lpthread`; `-static` on Windows.
- **pthreads/semaphores**: used for kernel mutexes (v2) and for the real concurrency demos (v1 `process_mgmt.c`). Windows builds rely on a winpthreads (MSYS2/MinGW) `pthread.h`.
- **Win32 API** (`comctl32`, `gdi32`, `comdlg32`, `-mwindows`) for the two native GUIs.
- **Python 3.8+ + Flask** for the web port (the only external runtime dependency; `pip install flask`). No `requirements.txt` / lockfile is present.
- **Build files:** `Makefile` (POSIX, builds **v1 only** → `taskforge`) and `build.bat` (Windows, hardcodes `C:\msys64\mingw64\bin`, builds v2/v1/gui/all/clean). There is **no CMake** and **no Makefile target for v2** — v2 is built only via `build.bat` or the manual `gcc` lines in the README.

---

## Build / run / test (actual commands)

Windows (MSYS2/MinGW on PATH — note `build.bat` sets it itself):
```bat
build.bat v2        :: -> taskforge_v2.exe (Banking on OS kernel, the main project)
build.bat v1        :: -> taskforge.exe    (v1 simulator)
build.bat gui       :: -> taskforge_gui.exe (v1 GUI; note: does NOT build the v2 GUI)
build.bat all
build.bat clean
```

POSIX (Makefile builds v1 only):
```bash
make            # -> ./taskforge   (v1 simulator)
make run
make clean
```

Manual v2 build (from README, verified against build.bat):
```bash
gcc -Wall -Wextra -std=c11 -Ikernel -c kernel/os_kernel.c -o obj/os_kernel.o
gcc -Wall -Wextra -std=c11 -Ikernel -c app/bank.c        -o obj/bank.o
gcc -Wall -Wextra -std=c11 -Ikernel -c main_v2.c         -o obj/main_v2.o
gcc -static -o taskforge_v2.exe obj/main_v2.o obj/os_kernel.o obj/bank.o -lpthread
```

Web UI (README's recommended demo):
```bash
pip install flask
python web/app.py      # http://localhost:5000
```

Prebuilt binaries are present in the working tree (`taskforge.exe`, `taskforge_v2.exe`, `taskforge_gui.exe`, `taskforge_gui_v2.exe`) but are gitignored/untracked.

---

## Tests

**There is no test suite.** No unit tests, no test framework, no CI config (`.github/`) anywhere in the tree. "Testing" is entirely manual/interactive: run the CLI/GUI/web app and exercise the menus. The kernel functions return status codes and log to the ring buffer, but nothing asserts on them programmatically. `build_err.txt` (empty, untracked, gitignored) is the only artifact resembling a build log.

---

## Status, completeness & notable gaps

- **Functional for its purpose** (a course demo). All three front-ends compile and run; prebuilt `.exe`s are checked into the working tree.
- **Not a real OS:** no bootloader, no kernel/user separation, no real context switching, no real address space — it is an in-process simulation. The README/title framing ("custom OS kernel," "runs through real system calls") is metaphorical; the "syscalls" are ordinary inlined C function calls.
- **Scheduler is dormant in v2.** `kernel_schedule_next` is fully implemented and runtime-selectable but is never invoked by the banking path — the bank manually transitions states. So changing the scheduler in the v2 CLI config menu changes the dashboard label but does not alter how a deposit executes. (The Python web port does call `schedule_next`.)
- **Resource-ordering prevention is OFF in C.** `kernel_init` sets `prevention_on = 0`, and nothing in the C path turns it on, yet `main_v2.c`'s boot banner and dashboard print "Banker's prevention: ON." Banker's *safety checking* still runs on every `sys_lock`; only the resource-ordering branch is inert. The web port defaults it to `True`.
- **Duplicated logic / drift risk.** The kernel exists three times (C kernel, Python kernel, and conceptually again in v1 `src/`), with small behavioral divergences (default priorities, prevention flag, whether scheduling runs). There is no shared source of truth.
- **Bank balance not round-tripped through the FS.** File reads happen for the trace but the authoritative balance stays in `BankState`; the serialized file content is never parsed back.
- **Fixed-capacity, single-instance** (20 accounts, one global `g_kernel`, no persistence — everything is lost on exit).
- **No license file**, despite being public.

---

## README vs. code

| README / CONTRIBUTIONS claim | Reality in code |
|---|---|
| "runs … through **real system calls**", "custom **OS kernel**" | Inlined C function calls into an in-process library; no kernel mode, no boot, no MMU. Accurate as a *simulation*, misleading taken literally. |
| Unit I: "**25+ system calls**" | 30 `static inline sys_*` wrappers in `os_syscall.h`. ✅ (slightly more than claimed) |
| Unit III lists **SRTF** and "preemptive & non-preemptive … Gantt charts and comparison mode" | True only in **v1 `src/scheduler.c`**. The **kernel** has FCFS/SJF/Priority/RR (non-preemptive, no Gantt) — and the bank never calls it. |
| Unit V: "paging, segmentation, **TLB** simulation … **Optimal** page replacement" | Only in **v1 `src/memory_mgmt.c`**. The kernel page cache implements LRU/FIFO/Clock only — no paging/segmentation/TLB/Optimal. |
| Unit IV: "**Resource Allocation Graph**", deadlock detection, "resource ordering prevention" | RAG (DFS cycle) is **v1 `deadlock.c`** only. Kernel has Banker's avoidance + detection. Resource ordering exists in the kernel but is **disabled** (`prevention_on = 0`) in the C path while the UI prints "ON". |
| Unit II: "real pthreads, … Producer-Consumer, Readers-Writers, Dining Philosophers demos" | Real pthreads/semaphores exist only in **v1 `process_mgmt.c`**; the v2 bank spawns no worker threads (kernel uses mutexes only). ✅ for v1. |
| "Resource lock … checks if the grant leaves the system in a safe state (Banker's)" | ✅ Verified — `kernel_resource_request` does tentative-allocate → `kernel_banker_safe` → rollback. |
| Transfers "lock both accounts using resource ordering (lower ID first)" | ✅ Verified in `bank_transfer` (and `BankApp.transfer`). |
| Sample trace shows "Memory allocated … (**Best Fit**)" and "Disk I/O … (**SCAN**)" | Illustrative — defaults are **First Fit** and **FCFS** disk (`kernel_init`); the strategy shown depends on current config. |
| Build commands (`build.bat`, manual gcc, Flask) | ✅ Match the actual `build.bat` and `Makefile`. Caveat: the **Makefile builds v1 only** (no v2 target), and `build.bat gui` builds the **v1** GUI, not `taskforge_gui_v2.c` (only the README's manual line builds the v2 GUI). |
| CONTRIBUTIONS "Total LOC: ~2,500 (kernel) + ~800 (banking) + ~600 (web)…" | Measured: kernel `os_kernel.c` 1,576 + headers 500 ≈ 2,076; `bank.c` 788; `web/app.py` 721. Ballpark-accurate. Whole repo C is ~12.5k LOC (dominated by the GUIs and v1). |

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\TaskForge\TaskForge. Source of truth: https://github.com/AmeyaBorkar/TaskForge-OS*
