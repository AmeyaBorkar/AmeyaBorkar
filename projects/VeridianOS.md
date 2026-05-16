# VeridianOS

> Bare-metal x86-64 microkernel using hardware virtualization (Intel VT-x / AMD-V) as its isolation primitive. ~51,000 lines of C and assembly. 214 automated tests on every boot. Boots on real Dell laptops.

**Repository:** [`AmeyaBorkar/VERIDIAN`](https://github.com/AmeyaBorkar/VERIDIAN)  
**Category:** Systems / OS  
**Visibility:** Private  
**Primary language:** C  
**Default branch:** `master`  
**License:** Apache License 2.0  
**Created:** 2026-04-03  
**Last pushed:** 2026-05-07  
**Metadata updated:** 2026-05-07  
**Size (GitHub reported):** 3,608 KB  

---

## What it is (one-paragraph version)

Bare-metal x86-64 microkernel using hardware virtualization (Intel VT-x / AMD-V) as its isolation primitive. ~51,000 lines of C and assembly. 214 automated tests on every boot. Boots on real Dell laptops.

## Language breakdown

| Language | Bytes | Share |
|----------|------:|------:|
| C | 2,774,303 | 97.9% |
| Assembly | 28,232 | 1.0% |
| Makefile | 21,136 | 0.7% |
| Shell | 7,561 | 0.3% |
| Linker Script | 2,878 | 0.1% |
| GDB | 292 | 0.0% |

## File tree

- Total entries indexed: **268** (245 files, 23 directories)

```
.gitignore  (308 B)
CONTRIBUTING.md  (6 KB)
LICENSE  (11 KB)
Makefile  (19 KB)
README.md  (23 KB)
build.sh  (1 KB)
linker.ld  (2 KB)
boot/    [2 files]
  boot/boot.asm
  boot/multiboot2.asm
docs/    [28 files]
  docs/8G_SMP_DIAGNOSIS_2026-04-23.md
  docs/AMD_NVME_DIAGNOSIS.md
  docs/AMD_USB_FIX.md
  docs/AUDIT_ADVERSARIAL_2026-04.md
  docs/AUDIT_CONVERGENCE_2026-04.md
  docs/AUDIT_FULL_SYSTEM_2026-04.md
  docs/AUDIT_FUNCTIONAL_2026-04.md
  docs/AUDIT_PHASE6_2026-04.md
  docs/AUDIT_PHASE7_2026-04.md
  docs/AUDIT_PREPUBLIC_2026-04.md
  docs/AUDIT_PRODUCTION_2026-04.md
  docs/AUDIT_USB_SAFETY_2026-04.md
  docs/AUDIT_WAVE_MANIFEST.md
  docs/BARE_METAL_MATRIX.md
  docs/BOOT_AUDIT.md
  ... and 13 more under docs/
domains/    [8 files]
  domains/common/hypercall.h
  domains/common/syscall.h
  domains/init/domain_init.c
  domains/init/domain_init.ld
  domains/shell/shell.c
  domains/shell/shell.ld
  domains/test/test_suite.c
  domains/test/test_suite.ld
grub/    [1 files]
  grub/grub.cfg
hypervisor/    [119 files]
  hypervisor/acpi.c
  hypervisor/acpi.h
  hypervisor/ahci.c
  hypervisor/ahci.h
  hypervisor/ap_trampoline.asm
  hypervisor/assert.c
  hypervisor/assert.h
  hypervisor/bcache.c
  hypervisor/bcache.h
  hypervisor/blk_dev.c
  hypervisor/blk_dev.h
  hypervisor/blk_recovery.c
  hypervisor/blk_recovery.h
  hypervisor/capability.c
  hypervisor/capability.h
  ... and 104 more under hypervisor/
include/    [7 files]
  include/config.h
  include/iommu_defs.h
  include/multiboot2.h
  include/svm_defs.h
  include/types.h
  include/vmx_defs.h
  include/x86.h
lib/    [4 files]
  lib/printf.c
  lib/printf.h
  lib/string.c
  lib/string.h
net/    [56 files]
  net/core/arp.c
  net/core/dhcp.c
  net/core/dns.c
  net/core/ethernet.c
  net/core/icmp4.c
  net/core/icmp6.c
  net/core/ip4.c
  net/core/ip6.c
  net/core/loopback.c
  net/core/net.c
  net/core/netport.c
  net/core/pbuf.c
  net/core/socket.c
  net/core/tcp.c
  net/core/udp.c
  ... and 41 more under net/
tools/    [3 files]
  tools/gdb_init.gdb
  tools/make_vbox_disk.sh
  tools/run_qemu.sh
userland/    [10 files]
  userland/Makefile
  userland/build-musl.sh
  userland/busybox/build-busybox.sh
  userland/busybox/veridian_defconfig
  userland/musl-patch/syscall_arch.h
  userland/tests/dirtest.c
  userland/tests/fileio.c
  userland/tests/hello.c
  userland/tests/malloctest.c
  userland/tests/net_smoke.c
```

## README (verbatim)

# VeridianOS

A bare-metal x86-64 microkernel that uses hardware virtualization as its isolation primitive.

VeridianOS repurposes Intel VT-x and AMD-V --- the CPU extensions that power virtual machines --- to isolate every OS component in its own hardware-enforced memory domain. Each domain runs inside nested page tables (EPT/NPT). A compromised domain cannot access another domain's memory; the CPU prevents it.

On top of this, the hypervisor provides transactional crash recovery, preemptive SMP scheduling, demand-paged variable-size domains, shared memory IPC, an ext2 filesystem with block-level transactions, a Linux-compatible syscall ABI with musl + busybox, and record/replay for deterministic debugging, all enforced through the same hardware mechanism.

~51,000 lines of C, headers, and assembly. No external dependencies. No libc in the kernel. Boots on real hardware.

---

## Tested Hardware

| Machine              | CPU                           | Cores | Virtualization | RAM    | Boot        | Tests (PASS / SKIP / FAIL) |
|----------------------|-------------------------------|-------|----------------|--------|-------------|----------------------------|
| Dell laptop          | Intel i7-11xxx (Tiger Lake)   | 8     | VMX + EPT      | 16 GB  | UEFI        | 106 / 43 / 0               |
| Dell laptop          | Intel i3-7100U (Kaby Lake)    | 4     | VMX + EPT      | 16 GB  | Legacy BIOS | 106 / 43 / 0               |
| QEMU + KVM           | AMD Ryzen AI 9 HX 370 host    | 4     | SVM + NPT      | 512 MB | BIOS        | 109 / 40 / 0               |
| QEMU TCG             | `qemu64` software             | 1     | Software       | 512 MB | BIOS        | 109 / 40 / 0               |

214 automated tests run on every boot (Phase 9 + audit-fix waves added 65 over the 8g.2 baseline).  The kernel checks results against a per-boot-mode baseline (`baremetal-safe`, `baremetal-storage`, `qemu-safe`, `qemu-storage`) and halts on any deviation. Release builds compile out assertions and fault-injection, yielding lower PASS counts against their own release baseline.

---

## Architecture

```
+-----------------------------------------------------------+
|                 User Domains (hardware-isolated)           |
|  +--------+  +--------+  +--------+  +--------+           |
|  | Domain | | Domain  | | Domain  | | Domain  |  ...      |
|  | 4MB-1GB| | 4MB-1GB | | 4MB-1GB | | 4MB-1GB |           |
|  | EPT/NPT| | EPT/NPT | | EPT/NPT | | EPT/NPT |           |
|  +--------+  +--------+  +--------+  +--------+           |
+-----------------------------------------------------------+
|  Syscall ABI (musl + busybox)  |  Capability-Gated         |
|  fork / exec / pipes / signals |  Hypercall Interface      |
+-----------------------------------------------------------+
|  Transaction Engine        |  Shared Memory IPC           |
|  (COW undo via EPT/NPT)    |  (grant / revoke / destroy)  |
+-----------------------------------------------------------+
|  ext2 FS + bcache + FS txn + versioning                   |
+-----------------------------------------------------------+
|  Record / Replay (RDTSC intercept, syscall log)           |
+-----------------------------------------------------------+
|  Preemptive Scheduler (per-CPU run queues, ticket locks)  |
+-----------------------------------------------------------+
|  Hardening: canaries, SMEP/SMAP, guard pages, NMI watchdog|
+-----------------------------------------------------------+
|  VMX / SVM  |  EPT / NPT  |  Buddy PMM  |  Slab kmalloc   |
+-----------------------------------------------------------+
|  IOAPIC  |  MSI/MSI-X  |  LAPIC Timer  |  IRQ Allocator   |
+-----------------------------------------------------------+
|  AHCI  |  NVMe  |  virtio-blk/net  |  xHCI/EHCI  |  PCI   |
+-----------------------------------------------------------+
|                    x86-64 Hardware                        |
+-----------------------------------------------------------+
```

---

## Key Ideas

### Hardware memory isolation

Every domain runs inside CPU-enforced nested page tables. Traditional process isolation relies on software-configured page tables that are vulnerable to kernel bugs. VeridianOS uses the same hardware that isolates virtual machines to isolate every OS component.

```
domain_id >= 0:    PASS
own_memory_mapped: PASS    -- GPA 0x200000 (own code) accessible
32mb_blocked:      PASS    -- GPA 0x2000000 (outside domain) BLOCKED by EPT
80mb_blocked:      PASS    -- GPA 0x5000000 BLOCKED by EPT
256mb_blocked:     PASS    -- GPA 0x10000000 BLOCKED by EPT
```

### Transactional crash recovery

Any domain can wrap operations in transactions at the OS level. The hypervisor write-protects all domain pages via EPT/NPT. On the first write to each page, the original content is copied to an undo log (copy-on-write). On commit, the log is discarded. On abort or crash, every modified page is restored.

Block-level filesystem transactions (`fstxn.c`) extend the same model to ext2 writes: a transaction batches inode, directory, and bitmap changes, commits atomically to the journal, or rolls back on failure.

```
txn_commit_persists:       PASS   -- changes survive commit
txn_abort_rollback:        PASS   -- changes reverted on abort
txn_multi_page_rollback:   PASS   -- multiple pages rolled back atomically
fstxn_commit_atomic:       PASS   -- ext2 inode + bitmap + block atomic
fstxn_crash_recovery:      PASS   -- partial txn detected and rolled back
```

### Demand-paged variable domains

Domains request 4 MB to 1 GB of address space. Physical pages are allocated only on first access via EPT/NPT fault handling. A 16 MB shell domain typically touches fewer than 10 physical pages.

### Shared memory IPC

Domains exchange data through grant-based shared memory regions mapped into both EPT/NPT tables. Measured latency: ~5 ns (17-27 cycles) through L1 cache.

### Linux-compatible syscall ABI

A single `syscall` entry point in the hypervisor services ~40 Linux syscalls on behalf of domains running musl-linked binaries. `fork`, `execve`, `wait4`, `pipe`, `kill`, file I/O, signals, and pipes all work. Busybox runs unmodified. `exec <file>` from the shell launches ELF binaries out of the ext2 filesystem.

### Record and replay

Every domain's execution can be recorded (syscall returns, RDTSC values, signal deliveries, preemption points). The replay engine re-runs the recorded log deterministically, reproducing bit-identical state. This is the foundation for the production time-travel debugger scheduled for Phase 13.

### Hardening

Phase 8 added stack canaries (`-fstack-protector-strong` with a per-boot random guard), SMEP/SMAP per-CPU, guard pages on every kernel stack, a dual-tick + scheduler-heartbeat NMI watchdog, block-driver recovery, a dmesg ring with offline dump via `savelog`, and a panic-capture crash dump. A 219-finding pre-production audit was closed across 60 fix commits.

---

## Performance

Measured on bare-metal Intel i3-7100U at 2.4 GHz:

| Operation                      | Latency    | Notes                             |
|--------------------------------|------------|-----------------------------------|
| `pmm_alloc_page`               | 52 ns      | Per-CPU page cache hit            |
| `pmm_free_page`                | 51 ns      | Per-CPU page cache hit            |
| `pmm_alloc_pages(512)` (2 MB)  | 45 ns      | Buddy bulk path                   |
| `kmalloc(64)`                  | 41 ns      | Slab magazine hit                 |
| `kfree(64)`                    | 39 ns      | Slab magazine hit                 |
| `kmap` 4 KB round-trip         | 280 ns     | Map + unmap + TLB invalidate      |
| TLB shootdown (4 CPUs)         | ~1850 ns   | IPI to 3 APs + ack                |
| Shared memory read             | ~5 ns      | L1 through EPT/NPT                |
| VM exit round-trip             | ~161 ns    | Hypercall path                    |

---

## Building

Prerequisites (Linux or WSL2):

```bash
sudo apt install make gcc nasm xorriso mtools grub-pc-bin grub-common \
    grub-efi-amd64-bin qemu-system-x86
```

Build and run:

```bash
make clean && make iso

# QEMU with KVM (recommended)
make run-kvm

# QEMU without KVM (software emulation)
make run

# Automated test suite (exits with 0 on baseline match)
make test
```

Bare metal:

```bash
# Write ISO to USB
sudo dd if=veridian.iso of=/dev/sdX bs=4M status=progress

# Boot from USB. Disable Secure Boot if needed.
# Tests run automatically; shell launches on success.
```

### Boot modes

GRUB presents three entries:

| Entry             | Default | Description                                                                 |
|-------------------|---------|-----------------------------------------------------------------------------|
| Safe Mode         | Yes     | Block-device writes disabled. Safe for machines with real data.             |
| Debug Mode        |         | Enables `verbose=1` --- full init output on screen and serial.              |
| Storage Test Mode |         | Unlocks block-device writes. Only for dedicated test disks.                 |

---

## What It Does

### Boot sequence

1. GRUB loads the kernel via Multiboot2 (legacy BIOS or UEFI).
2. 32-bit to 64-bit transition; 4 GB identity-mapped page tables.
3. Serial, VGA, and framebuffer console initialization (including above-4 GB UEFI framebuffers with PAT + write-combining).
4. Command line parsing (`safe_mode`, `disk_write`, `verbose`, `fb=off`, `spec_mit=off`, others).
5. Buddy PMM initialization (3 zones, per-CPU page caches, up to 1 TB).
6. IDT with 32 exception handlers and IST1 stacks per CPU.
7. Kernel page tables with W^X enforcement and IPI-based TLB shootdown.
8. SMP startup --- all cores online via INIT/SIPI, ACPI APIC topology.
9. Slab allocator (16 size classes, per-CPU magazines).
10. IOAPIC + IRQ vector allocation + LAPIC timer (periodic ~10 ms).
11. PCI/PCIe enumeration with ECAM, bridge recursion, MSI/MSI-X capability walking.
12. Storage drivers: AHCI (SATA), NVMe, virtio-blk --- all interrupt-driven.
13. USB drivers: xHCI (USB 3.0), EHCI (USB 2.0) with BIOS handoff; USB-MSC for mass storage.
14. Filesystem mount with foreign-FS detection; ext2 with block cache, transactions, versioning.
15. Hardening activation: SMEP/SMAP, stack guards, NMI watchdog, spec-mit MSRs.
16. Hypervisor initialization (VMX or SVM, auto-detected).
17. AP scheduler activation --- all CPUs enter their scheduler loops.
18. 214 automated tests inside an isolated domain, with baseline verification.
19. Screen clear, boot summary, interactive shell (busybox on ext2).

### Domain lifecycle

```
CREATE  ->  Allocate physical memory (scatter-mapped, demand-paged)
        ->  Build EPT/NPT (GPA -> HPA)
        ->  Initialize VMCB (AMD) or VMCS (Intel)
        ->  Grant capabilities (MEMORY, IO, IPC, DOMAIN)

RUN     ->  VMLAUNCH/VMRUN enters guest
        ->  EPT/NPT enforces memory boundaries
        ->  Timer fires -> VM exit -> preempt if quantum expired
        ->  Hypercall -> capability check -> execute or deny -> resume
        ->  Syscall -> Linux-compatible handler -> resume

FORK    ->  Copy-on-write EPT/NPT fan-out
        ->  Parent + child share physical pages until first write
        ->  Each write faults, clones page, patches child's EPT

DESTROY ->  Rollback active transactions
        ->  Free VMCB/VMCS, page tables, physical memory
```

### Transaction flow

```
txn_begin   ->  Clear write bit on every domain page in EPT/NPT
            ->  Flush TLB

Write fault ->  Copy original page to undo log
            ->  Mark page writable, resume

txn_commit  ->  Discard undo log (O(1))
txn_abort   ->  Restore pages from undo log (O(n modified pages))
```

### Security model

Every privileged hypercall passes through capability checks. Domains hold typed access tokens (MEMORY, IO, IPC, DOMAIN) in a table that lives in hypervisor memory. Domains cannot forge, modify, or escalate capabilities.

```
Domain A                         Hypervisor                    Domain B
   |                                |                             |
   |-- VMMCALL(SPAWN) ------------->|                             |
   |                                |-- check DOMAIN+EXEC cap     |
   |                                |-- domain_create()           |
   |<-- domain ID ------------------|                             |
   |                                |                             |
   |-- read GPA 0x2000000 --------->|                             |
   |                                |-- EPT: NOT MAPPED           |
   |<-- #EPT violation, killed -----|                             |
```

See [docs/SECURITY_MODEL.md](docs/SECURITY_MODEL.md) for the full capability + mitigation model.

---

## Shell

```
veridian> help
  Core
    help                    Show this help
    info                    System information
    mem                     Memory statistics
    caps                    List domain capabilities
    ticks                   LAPIC tick counter
    spawn                   Spawn a new isolated domain
    ps                      List all domains
    halt                    Shutdown system

  Filesystem (ext2)
    ls [path]               List files
    cat <file>              Print file contents
    write <n> <data>        Write data to file
    read <file>             Read file contents
    rm <file>               Delete a file
    mkdir <path>            Create directory
    rmdir <path>            Remove directory
    touch <file>            Create empty file
    mkfs <dev>              Format device as ext2
    mount [dev]             Mount ext2 filesystem
    stat <file>             File metadata
    sync                    Flush bcache to disk
    fsck                    Filesystem integrity check
    vfs format              Format disk (typed confirmation)

  Transactions + versioning
    txn <sub>               Transaction control
    versions <file>         List file versions
    revert <file> <ver>     Roll file back to version

  Execution
    exec <file>             Load + run ELF binary from ext2

  Test / bench
    test isolation          EPT/NPT memory isolation test
    test txn                Transaction commit/rollback test
    bench                   VM exit latency benchmark
    bench pmm               PMM allocator benchmark
    bench kmalloc-64        Heap 64B hot path
    bench kmalloc-4k        Heap large (>4K) path
    bench kmap-4k           kmap 4KB MMIO latency
    bench kmap-2mb          kmap 2MB latency
    bench kmap-1gb          kmap 1GB latency
    bench tlb-flush-1       TLB shootdown 1 page
    bench tlb-flush-32      TLB shootdown 32 pages

  Networking
    net                     Network device info
    ping [ip]               Ping (default: 10.0.2.2)

  Diagnostics
    pci                     List PCI devices
    disk                    Disk info + read sector 0
    ahcidebug               AHCI state dump
    nvmedebug               NVMe state dump
    mouse                   USB mouse event test
    xray <sub>              Diagnostic dump
                            (pmm|pci|cpu|ahci|nvme|vfs|domain|
                             blk|kmap|heap|irq|sched|all)
    dmesg [n]               Print last N log entries
    savelog <path>          Dump log ring to ext2 file
    crashdump [path]        Emit crash dump

  Record / replay
    record <dom>            Start recording a domain
    stoprec <dom>           Stop recording
    replay <rec>            Replay a recorded run
    recinfo <rec>           Recording metadata

  Hardening / fault injection
    fuzz <n>                Run syscall fuzzer for N iterations
    stress <n>              Run slab + PMM stress for N iterations
    injectfail [sub]        Inject fault into block stack
    echo <text>             Print text
```

---

## Project Structure

```
VeridianOS/
├── boot/
│   ├── multiboot2.asm             Multiboot2 header
│   └── boot.asm                   32->64 bit transition, initial page tables
├── hypervisor/
│   ├── main.c                     Boot sequence, test orchestration
│   ├── vmx.c / svm.c              Intel VT-x / AMD-V hypervisor
│   ├── vmx_asm.asm / svm_asm.asm  VMLAUNCH/VMRUN register save/restore
│   ├── ept.c                      EPT/NPT page table management
│   ├── domain.c                   Domain lifecycle, demand paging, shared memory
│   ├── txn.c                      Page-level transactional crash recovery
│   ├── fstxn.c                    Filesystem-level transactions
│   ├── sched.c                    Preemptive per-CPU scheduler
│   ├── capability.c               Capability-based access control
│   ├── syscall.c                  Linux-compatible syscall ABI
│   ├── replay.c                   Record / replay engine
│   ├── elf.c                      ELF64 loader with bounds checking
│   ├── pmm.c                      Buddy allocator (3 zones, per-CPU caches)
│   ├── kmap.c                     Kernel page tables, W^X, TLB shootdown
│   ├── kmalloc.c                  Slab allocator, per-CPU magazines
│   ├── smp.c                      SMP boot, AP entry, ticket locks
│   ├── acpi.c                     RSDP/MADT parsing
│   ├── pci.c                      PCI/PCIe enumeration, ECAM, MSI/MSI-X
│   ├── ahci.c                     SATA driver (MSI, interrupt-driven)
│   ├── nvme.c                     NVMe driver (MSI/MSI-X, interrupt-driven)
│   ├── virtio_blk.c / virtio_net.c  QEMU virtio drivers
│   ├── usb/xhci.c / usb/ehci.c    USB 3.0 / 2.0 host controllers
│   ├── usb/usb_msc.c              USB mass-storage class driver
│   ├── usb/usb_mouse.c            USB HID mouse driver
│   ├── irq.c / ioapic.c           IRQ vector allocator, IOAPIC driver
│   ├── framebuffer.c              UEFI pixel console + PAT/WC mapping
│   ├── vga.c / serial.c           VGA text + COM1 serial console
│   ├── vfs.c                      VFS layer
│   ├── ext2.c                     ext2 filesystem (read + write + versions)
│   ├── bcache.c                   Block cache
│   ├── blk_dev.c                  Block-device abstraction
│   ├── blk_recovery.c             Driver-level recovery + fault injection
│   ├── gpt.c                      GPT partition table parser
│   ├── assert.c                   VERIDIAN_ASSERT runtime
│   ├── fuzz.c                     HCALL + syscall fuzzer
│   ├── stress.c                   Slab + PMM stress harness
│   ├── fault_inject.c             Fault-injection framework
│   ├── stack_protect.c            Stack canary support
│   ├── stack_guard.c              Per-CPU stack guard pages
│   ├── watchdog.c                 NMI watchdog (tick + sched heartbeats)
│   ├── crashdump.c                Panic capture + serialisation
│   ├── log_ring.c                 dmesg ring buffer
│   ├── cpu_features.c             CPUID, PAT, spec-mit MSRs
│   ├── cmdline.c                  Boot command line parser
│   ├── timer.c                    LAPIC timer + PIT fallback
│   ├── idt.c / idt_asm.asm        Interrupt descriptor table + stubs
│   ├── ap_trampoline.asm          AP SIPI entry
│   ├── test_safety.c              214 kernel-side safety tests + baselines
│   └── xray.c                     Runtime diagnostics (12 subsystems)
├── domains/
│   ├── common/                    Shared headers (hypercall.h, syscall.h)
│   ├── init/                      Init domain
│   ├── test/                      Test domain
│   └── shell/                     Interactive shell
├── userland/
│   ├── musl-patch/                Port of musl libc to VeridianOS syscalls
│   ├── busybox/                   Busybox build config
│   └── tests/                     Userland test binaries
├── lib/
│   ├── string.c                   memcpy, memset, strcmp, strlen
│   └── printf.c                   kprintf family
├── include/
│   ├── types.h                    Base types, bool, PACKED, ALIGNED
│   ├── config.h                   Build-mode flags (debug vs release)
│   ├── x86.h                      Port I/O, MSRs, CPUID, CR registers
│   ├── vmx_defs.h / svm_defs.h    VMCS/VMCB layouts, exit codes, hypercalls
│   └── multiboot2.h               Multiboot2 structures
├── Makefile                       Build system (-O2, -Werror, -mno-sse)
├── linker.ld                      Kernel at 1 MB physical
├── grub/grub.cfg                  GRUB menu (Safe / Debug / Storage Test)
├── docs/ROADMAP.md                Development roadmap
├── CONTRIBUTING.md                Build instructions and design rules
└── LICENSE                        Apache 2.0
```

---

## Roadmap

| Phase | Status | Description                                                                 |
|-------|--------|-----------------------------------------------------------------------------|
| 0     | Done   | Boot safety, block-device write protection, foreign-FS detection            |
| 1     | Done   | Buddy PMM, kernel page tables (kmap/W^X/TLB shootdown), slab allocator      |
| 2     | Done   | IOAPIC, MSI/MSI-X, interrupt-driven NVMe + AHCI                             |
| 3     | Done   | Preemptive scheduler, per-CPU run queues, SMP, ticket locks                 |
| 4     | Done   | Full PCI/PCIe enumeration, ECAM, bridge recursion, BAR management           |
| 5     | Done   | Variable domain memory, demand paging, shared memory IPC                    |
| 6     | Done   | ext2 filesystem, block cache, FS transactions, versioning                   |
| 7     | Done   | Syscall ABI, musl libc, fork/exec, busybox shell, record/replay             |
| 8     | Done   | Hardening (canaries, SMEP/SMAP, guard pages, watchdog, crash dump)          |
| 9     | Next   | Network stack (TCP/UDP, DHCP, DNS, sockets) --- ships as VeridianOS 1.0     |

See [docs/ROADMAP.md](docs/ROADMAP.md) for the full 12-month plan through 3.0 GA.

---

## Research Context

VeridianOS combines ideas from several areas of systems research:

- Microkernel isolation (seL4, L4) --- domains communicate via IPC, not shared kernel state.
- Hardware isolation (Xen, KVM) --- CPU virtualization as the component boundary.
- Transactional memory (Intel TSX) --- applied at the OS page level via EPT/NPT write-protection.
- Record / replay (rr, VMware) --- native in the hypervisor; foundation for production time-travel debugging.
- Preemptive multikernel (Barrelfish) --- per-CPU run queues with IPI-based coordination.
- Capability systems (KeyKOS, EROS, seL4) --- every privileged hypercall gated by unforgeable tokens.
- Linux-ABI compatibility (gVisor, Firecracker) --- ~40 syscalls, enough to run musl + busybox.

The contribution is unifying EPT/NPT-based isolation with transactions, demand paging, preemptive per-CPU scheduling, a Linux syscall ABI, and deterministic record/replay in a single architecture where the same hardware mechanism provides isolation, enables transactions, and underpins scheduling.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

## Author

Ameya Borkar

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/VERIDIAN*
