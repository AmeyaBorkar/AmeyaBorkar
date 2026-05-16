# SyncUp

> A concurrent file backup / sync utility in C that is secretly a full Operating Systems course in disguise — every topic (scheduling, paging, deadlocks, disk I/O) simulated interactively.

**Repository:** [`AmeyaBorkar/SyncUp-Concurrent-File-Backup-Sync-System`](https://github.com/AmeyaBorkar/SyncUp-Concurrent-File-Backup-Sync-System)  
**Category:** Systems / OS (educational)  
**Visibility:** Public  
**Primary language:** C  
**Default branch:** `main`  
**License:** —  
**Created:** 2026-03-24  
**Last pushed:** 2026-03-24  
**Metadata updated:** 2026-03-24  
**Size (GitHub reported):** 212 KB  

---

## What it is (one-paragraph version)

A concurrent file backup / sync utility in C that is secretly a full Operating Systems course in disguise — every topic (scheduling, paging, deadlocks, disk I/O) simulated interactively.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| C | 210,016 | 97.6% |
| Makefile | 5,177 | 2.4% |

## File tree

- Total entries indexed: **35** (33 files, 2 directories)

```
Makefile  (5 KB)
README.md  (9 KB)
compile_errors.txt  (3 KB)
syncup.exe  (422 KB)
include/    [14 files]
  include/deadlock_detector.h
  include/disk_scheduler.h
  include/file_io.h
  include/file_system_sim.h
  include/memory_pool.h
  include/os_concepts.h
  include/process_manager.h
  include/process_states.h
  include/scheduler.h
  include/synchronization.h
  include/syscall_wrappers.h
  include/thread_pool.h
  include/utils.h
  include/virtual_memory.h
src/    [15 files]
  src/deadlock_detector.c
  src/disk_scheduler.c
  src/file_io.c
  src/file_system_sim.c
  src/main.c
  src/memory_pool.c
  src/os_concepts.c
  src/process_manager.c
  src/process_states.c
  src/scheduler.c
  src/synchronization.c
  src/syscall_wrappers.c
  src/thread_pool.c
  src/utils.c
  src/virtual_memory.c
```

## README (verbatim)

# SyncUp — Concurrent File Backup & Sync System

> **Advanced Operating Systems Course Project**
> A comprehensive, real-world file backup utility written in C that practically demonstrates **every topic from all 6 units** of an extensive Operating Systems syllabus.

---

## 🎯 Project Overview

SyncUp is not just a theoretical simulation, but a fully functional file synchronization and backup tool that integrates core OS concepts directly into its architecture. The project runs in two primary modes:

1. **Backup Mode (`./syncup <src> <dest>`)**: A high-performance, concurrent directory mirror. It uses multiple processes to handle subdirectories independently, thread pools for parallel file copying within those directories, custom memory allocators to prevent fragmentation during intense I/O bounds, and strict deadlock detection algorithms to safely manage file locks.
2. **Demo Mode (`./syncup --demo <unit>`)**: An educational suite of interactive simulations that run directly in the terminal, visualizing abstract OS concepts like Gantt chart scheduling, memory fragmentation, disk head movement traces, and resolving classical synchronization problems in real-time.

---

## 📚 Complete Syllabus Coverage & Implementation

### Unit I — Introduction to Operating Systems
This unit explores the fundamental types and services of operating systems.
- **Types of OS Simulated**: The project actively simulates Batch, Multiprogramming, Time-sharing, Parallel, and Real-Time systems utilizing a uniform task set. It showcases standard metrics comparing throughput versus response times.
- **Implementation**: Managed in `os_concepts.c`. Use `--demo 1` to observe how different OS models juggle the same CPU/IO burst workloads differently.
- **System Calls**: Core backup operations directly utilize lower-level POSIX system calls wrapped safely to handle `EINTR` retry semantics (`syscall_wrappers.c`).

### Unit II — Process Management & Sync Problems
Process life cycle, process description, and classical synchronization problems.
- **Process States**: Simulates 2-state, 5-state (New, Ready, Running, Waiting, Terminated) and 7-state models (adds suspended states). Tracked via Custom Process Control Blocks (PCBs) visualizing process transitions.
- **Orchestration**: The backup system leverages `fork()` to spawn robust worker processes per directory, communicating progress over pipe-based Inter-Process Communication (IPC).
- **Classical Sync Problems Implemented**:
  - *Producer-Consumer*: Implemented using custom counted Semaphores and a bounded buffer for scanner/worker threads.
  - *Readers-Writers*: Implemented with `pthread_mutex` and condition variables for shared access patterns.
  - *Dining Philosophers*: Deadlock-free implementation using strict Resource Ordering logic.
- **Implementation**: Handled in `process_states.c`, `process_manager.c`, and `synchronization.c`. Demo via `--demo 2`.

### Unit III — Process Scheduling
Visualizing and benchmarking CPU dispatch algorithms.
- **Algorithms**: First-Come-First-Serve (FCFS), Shortest Job First (SJF Non-preemptive & SRTF Preemptive), Round Robin (configurable quantum), and Priority Scheduling (NP & P).
- **Execution**: The scheduler computes the Start Time, Completion Time, Turnaround Time, Waiting Time, and Response Time for each simulated process, and subsequently produces a text-based Gantt chart to visualize execution sequences.
- **Implementation**: Found in `scheduler.c`. Accessible via `--demo 3`.

### Unit IV — Deadlocks
Strategies for managing and averting system deadlocks.
- **Detection**: Live Resource Allocation Graph (RAG) mapped globally. A background thread performs Depth-First Search (DFS) cycle-detection.
- **Prevention**: Enforces lock ordering by strict alphabetical filepath sorting before acquisition (Circular-wait prevention).
- **Avoidance**: Integrates the **Banker's Algorithm**, requiring processes to declare max needs, granting resources only if it results in a "Safe State".
- **Recovery**: Processes wait-times are benchmarked; victim processes are probabilistically selected and preempted if blocking exceeds threshold timeouts.
- **Implementation**: See `deadlock_detector.c`. Execute `--demo 4`.

### Unit V — Memory Management
Efficient allocation techniques and virtual memory operations.
- **Custom Memory Allocator**: To prevent `malloc` overhead during file copying, SyncUp uses a Slab Allocator maintaining 4KB, 16KB, and 64KB free-lists for near-instant memory provisioning in isolated pools.
- **Continuous Allocation Demos**: First Fit, Best Fit, Next Fit, and Worst Fit algorithms responding to external fragmentation challenges. Includes **Buddy System** (power-of-2 block splitting/coalescing) demos tracking internal fragmentation anomalies.
- **Virtual Memory & Paging**: Complete simulation containing Page Tables, configurable Translation Lookaside Buffers (TLB), FIFO, LRU, Optimal, and Clock Replacement Policies, plus *Thrashing Detection* based on high-frequency fault rates.
- **Implementation**: See `memory_pool.c` and `virtual_memory.c`. Trigger `--demo 5`.

### Unit VI — I/O and File Management
Secondary storage dispatching and file organization logic.
- **Disk Scheduling Algorithms**: FCFS, Shortest Seek Time First (SSTF), SCAN (Elevator), and C-SCAN scheduling. Demonstrates path-seek visualization and average-movement calculations.
- **File System Simulation**: Simulated directory trees featuring Inodes holding mapping schemes. Explores Continuous (Sequential), Direct, and Indexed file allocation typologies alongside tracking via Free Space Bitmaps.
- **Practical Operation**: Core code implements Buffered Read/Write cycles natively utilizing chunk-sizing matching slab-pool classes across the storage devices.
- **Implementation**: Covered in `disk_scheduler.c` and `file_system_sim.c`. Execute `--demo 6`.

---

## 🏗️ Project Architecture Breakdown

```
OperatingSystemCourseProject/
├── Makefile
├── README.md
├── include/              (14 Component Headers)
│   ├── deadlock_detector.h
│   ├── disk_scheduler.h
│   ├── file_io.h
│   ├── file_system_sim.h
│   ├── memory_pool.h
│   ├── os_concepts.h
│   ├── process_manager.h
│   ├── process_states.h
│   ├── scheduler.h
│   ├── synchronization.h
│   ├── syscall_wrappers.h
│   ├── thread_pool.h
│   ├── virtual_memory.h
│   └── utils.h
└── src/                  (15 Implementation Sources)
    ├── main.c              ← Command entry, CLI args & phase orchestration
    ├── os_concepts.c       ← Unit I logic
    ├── process_states.c    ← Unit II simulation
    ├── process_manager.c   ← Unit II practical multiprocessing
    ├── synchronization.c   ← Unit II POSIX implementations
    ├── scheduler.c         ← Unit III CPU simulation logic
    ├── syscall_wrappers.c  ← Robust generic wrappers
    ├── thread_pool.c       ← Queue priorities & thread routing
    ├── deadlock_detector.c ← Unit IV Graph handling & algorithms
    ├── memory_pool.c       ← Unit V Practical caching & allocation logic
    ├── virtual_memory.c    ← Unit V TLB & paging abstractions
    ├── disk_scheduler.c    ← Unit VI Drive-head calculation simulation
    ├── file_system_sim.c   ← Unit VI Record abstraction logic
    ├── file_io.c           ← Recursive walking & sync writing
    └── utils.c             ← Metrics & hashing logic
```

---

## 🔨 Build & Quick Start Guide

### Prerequisites
- GCC Compiler (Windows with MSYS2/MinGW, Linux, macOS, WSL)
- Basic POSIX threading support

### Windows Compilation Note
If compiling on Windows and using an MSYS2 GCC Wrapper, simply utilize:
```cmd
gcc src\*.c -Iinclude -pthread -o syncup.exe
```
Or if using Make:
```bash
make
```

### Running SyncUp Demos
The easiest way to understand the project architecture in relation to the syllabus is through the live demos:
```bash
# General OS Concept Types
./syncup --demo 1

# Process Scheduling Gantt Demos
./syncup --demo 3

# Memory Placement & Page Eviction Demos
./syncup --demo 5

# Run the entire simulation suite end-to-end
./syncup --demo all
```

### Running Practical Backups
Mirror files from your documents to an archive directory using 8 threads and verbose logging:
```bash
./syncup C:\Path\To\Source C:\Path\To\Dest -t 8 -v
```

---

## 📝 Academic Integrity & Licensing
This application was synthesized as an educational proof-of-concept exploring system-level architecture constraints mapping theoretical Operating System components into functioning application models. Free to utilize or reference for course studies or general reference.

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/SyncUp-Concurrent-File-Backup-Sync-System*
