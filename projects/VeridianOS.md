# VeridianOS

> Bare-metal x86-64 type-1 hypervisor microkernel using CPU virtualization (Intel VT-x / AMD-V) plus IOMMU (VT-d / AMD-Vi) as its isolation primitives. **~70,400 lines of C and assembly across 247 source files**, including a full TCP/IP networking stack, ext2 filesystem with journal-style transactions, Linux-compatible syscall ABI, and deterministic record/replay. Boots end-to-end on three machines spanning Intel Kaby Lake (2017), Tiger Lake (2020), and AMD Zen 5 (2024).

**Repository:** [`AmeyaBorkar/VERIDIAN`](https://github.com/AmeyaBorkar/VERIDIAN)  
**Category:** Systems / OS  
**Visibility:** Private  
**Primary language:** C  
**Default branch:** `master`  
**License:** Apache License 2.0  
**Created:** 2026-04-03  
**Last pushed:** 2026-05-16  
**Metadata updated:** 2026-05-16  
**Size (GitHub reported):** 3,608 KB  

---

## What it is (one-paragraph version)

Bare-metal x86-64 type-1 hypervisor microkernel using CPU virtualization (Intel VT-x / AMD-V) plus IOMMU (VT-d / AMD-Vi) as its isolation primitives. **~70,400 lines of C and assembly across 247 source files**, including a full TCP/IP networking stack, ext2 filesystem with journal-style transactions, Linux-compatible syscall ABI, and deterministic record/replay. Boots end-to-end on three machines spanning Intel Kaby Lake (2017), Tiger Lake (2020), and AMD Zen 5 (2024).

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

- Total entries indexed: **270** (247 files, 23 directories)

```
.gitignore  (389 B)
CONTRIBUTING.md  (6 KB)
LICENSE  (11 KB)
Makefile  (19 KB)
README.md  (30 KB)
build.sh  (1 KB)
linker.ld  (2 KB)
boot/    [2 files]
  boot/boot.asm
  boot/multiboot2.asm
docs/    [30 files]
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
  ... and 15 more under docs/
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

A bare-metal x86-64 type-1 hypervisor microkernel that uses CPU virtualization extensions (Intel VT-x and AMD-V) as its primary OS isolation mechanism.

Every OS component — drivers, filesystem, shell, applications, test suite — runs inside its own hardware-isolated *domain*. Each domain has dedicated nested page tables (Intel EPT or AMD NPT). A compromised domain cannot read another domain's memory because the CPU's MMU physically does not translate the address. This is the same hardware that isolates customer VMs in cloud platforms, applied at process granularity instead of VM granularity.

On top of this isolation primitive the hypervisor provides:

- Preemptive SMP scheduling with per-CPU run queues and ticket locks
- Demand-paged variable-size domains (4 MB to 1 GB) with scatter mapping
- Capability-gated hypercall and syscall interface — no root, no privilege escalation paths
- Transactional crash recovery via EPT/NPT write-protection + COW undo log
- ext2 filesystem with block cache, journal-style transactions, and snapshot versioning
- Linux-compatible syscall ABI (~50 syscalls) running musl-linked busybox
- Deterministic record / replay of any domain's execution
- IOMMU-enforced device isolation (Intel VT-d + AMD-Vi) with interrupt remapping

**70,398 lines of C, headers, and assembly across 199 source files. No external dependencies. No libc inside the kernel. Boots on three different bare-metal machines spanning two CPU vendors and three CPU generations (Kaby Lake 2017, Tiger Lake 2020, Zen 5 2024).**

---

## Tested Hardware

| Machine          | CPU                          | Cores | Virtualization | RAM    | Boot        | Status |
|------------------|------------------------------|-------|----------------|--------|-------------|--------|
| Dell laptop      | Intel i3-7100U (Kaby Lake)   | 4     | VMX + EPT      | 16 GB  | Legacy BIOS | ✅ end-to-end verified |
| Dell laptop      | Intel i7-11xxx (Tiger Lake)  | 8     | VMX + EPT      | 16 GB  | UEFI        | ✅ end-to-end verified |
| ASUS ROG Zephyrus| AMD Ryzen AI 9 HX 370 (Zen 5)| 12    | SVM + NPT      | 32 GB  | UEFI        | ✅ end-to-end verified |
| QEMU + KVM       | Intel host / AMD host        | 4     | VMX/SVM passthrough | 512 MB | BIOS+UEFI | ✅ baseline + regression |
| QEMU TCG         | `qemu64` software            | 1-4   | software emulation | 512 MB | BIOS    | ✅ baseline |

Three bare-metal platforms span legacy BIOS, modern UEFI, and AMD UEFI; two CPU vendors; four CPU generations from 2017 to 2024. The kernel runs hundreds of automated tests on every boot against per-platform baselines and halts on any deviation. See [docs/BARE_METAL_MATRIX.md](docs/BARE_METAL_MATRIX.md) for the full per-feature support matrix.

---

## Architecture

```
+---------------------------------------------------------------+
|             User Domains (hardware-isolated)                  |
|  +--------+ +--------+ +--------+ +-----------------+         |
|  | shell  | | editor | | vault  | | Linux guest     |         |
|  | native | | native | | native | | full Linux +    |  ...    |
|  | EPT/NPT| | EPT/NPT| | EPT/NPT| | userland        |         |
|  +--------+ +--------+ +--------+ +-----------------+         |
+---------------------------------------------------------------+
|  Linux Syscall ABI (musl/busybox)  |  Capability-Gated         |
|  fork/exec/pipes/signals           |  Hypercall Interface      |
+---------------------------------------------------------------+
|  Transaction Engine          |  Shared Memory IPC             |
|  (COW undo via EPT/NPT)      |  (grant / revoke / destroy)    |
+---------------------------------------------------------------+
|  ext2 + block cache + FS journal + snapshot versioning        |
+---------------------------------------------------------------+
|  Record / Replay (RDTSC intercept, syscall log)               |
+---------------------------------------------------------------+
|  Preemptive Scheduler (per-CPU run queues, ticket locks)      |
+---------------------------------------------------------------+
|  Hardening: canaries, SMEP/SMAP, guard pages, NMI watchdog,   |
|  IBPB / L1D_FLUSH / VERW spec mitigations                     |
+---------------------------------------------------------------+
|  VMX / SVM   |  EPT / NPT  |  Buddy PMM   |  Slab kmalloc     |
+---------------------------------------------------------------+
|  IOMMU (Intel VT-d + AMD-Vi)  |  Interrupt Remapping          |
+---------------------------------------------------------------+
|  IOAPIC | MSI/MSI-X | LAPIC Timer | x2APIC | IRQ Allocator    |
+---------------------------------------------------------------+
|  AHCI | NVMe | virtio-blk/net | xHCI/EHCI | USB-MSC | PCI/PCIe|
+---------------------------------------------------------------+
|                       x86-64 hardware                         |
+---------------------------------------------------------------+
```

For the long-form architecture (native vs Linux-guest domains, hybrid composition, IPC layers), see [docs/DOMAIN_ARCHITECTURE.md](docs/DOMAIN_ARCHITECTURE.md).

---

## Key Ideas

### Hardware-enforced memory isolation

Every domain runs in its own nested page table tree (EPT on Intel, NPT on AMD). Traditional process isolation depends on software-managed page tables enforced by a single shared kernel — a kernel bug compromises every process. VeridianOS uses the same hardware that isolates customer VMs in cloud platforms, applied at process granularity. There is no shared kernel that domains can attack; the hypervisor is unreachable from any domain except via explicit, capability-gated hypercalls.

```
domain_id >= 0:    PASS
own_memory_mapped: PASS    -- own code at GPA 0x200000 accessible
32mb_blocked:      PASS    -- GPA outside domain blocked by EPT
80mb_blocked:      PASS    -- larger GPA also blocked
256mb_blocked:     PASS    -- and at much larger offsets too
```

### Capability-based access control

There is no root, no sudo, no UID 0 to escalate to. Every privileged operation is gated by a capability the hypervisor holds on behalf of the domain. Capabilities are typed (MEMORY, IO, IPC, DOMAIN, NETWORK, DEBUG), unforgeable from inside a domain, and time-limited where appropriate. A domain can do exactly what it was granted at spawn time and nothing more.

### Transactional crash recovery

Any domain can wrap operations in a transaction. The hypervisor write-protects every domain page through EPT/NPT. On the first write to each page, the original contents are copied to an undo log; the write proceeds. On `txn_commit` the log is discarded (O(1)). On `txn_abort` or crash, every modified page is restored from the log (O(n modified pages)). The same mechanism extends to filesystem-level transactions: ext2 metadata, directory entries, and bitmap changes are batched and committed atomically through a journal, with automatic rollback on crash detected at the next mount.

### Demand-paged variable domains

Domains declare a target address space size between 4 MB and 1 GB. Physical pages are allocated only on first access via EPT/NPT fault handling. A 16 MB shell domain typically touches fewer than 10 physical pages, with the rest of its address space backed by nothing until it is needed.

### Shared memory IPC

Domains exchange data through hypervisor-mediated shared memory regions. The owner allocates a region; the hypervisor grants the receiver an EPT/NPT mapping with explicit RW or RO flags and tracks the region in software-available bits of the leaf entries. Either side can revoke. Measured latency for a shared-memory load: ~5 ns (17–27 cycles) through L1 cache.

### Linux-compatible syscall ABI

A single hypervisor syscall entry point services ~50 Linux syscalls on behalf of domains running musl-linked binaries. `fork`, `execve`, `wait4`, `pipe`, `kill`, file I/O, `mmap`, signals, sockets, `poll`, and the rest of the busybox syscall surface all work. Domain binaries are built against an unmodified musl source tree (with a small VeridianOS port for syscall numbers) and busybox runs unmodified. `exec /bin/ls` from the shell loads a real ELF binary out of the ext2 filesystem.

### Network stack

A from-scratch TCP/IP stack runs as a hypervisor library: ARP, IPv4, IPv6, ICMP, UDP, TCP with SACK + DSACK + Reno congestion control, DHCP client, DNS resolver, BSD sockets, `poll`. virtio-net is the production driver path with MSI-X and soft-IRQ-based RX delivery. busybox `wget` and `nc` run end-to-end against the stack inside a domain. See [Phase 9 work for details].

### Record and replay

Every domain's execution can be recorded: VM exits, syscall arguments + returns, RDTSC values, signal deliveries, preemption points, IO port values. The replay engine reproduces execution deterministically with bit-identical state — a foundation for production-grade time-travel debugging.

### IOMMU-enforced device isolation

Intel VT-d and AMD-Vi DMA remapping plus interrupt remapping. Every PCIe device is auto-attached at boot. Device DMA stays within whatever memory the hypervisor permitted; a misbehaving device cannot DMA into hypervisor memory or another domain's memory. Fault recording (Intel) and the Event Log (AMD) surface mis-routed DMA attempts as observable events. With IOMMU, hardware passthrough becomes safe — a Linux guest can be given exclusive ownership of a physical GPU or NIC without breaking the isolation invariant.

### Hardening

The hypervisor ships with: stack canaries (`-fstack-protector-strong` with per-boot random guard from RDSEED), SMEP/SMAP per CPU, kernel stack guard pages, IST stacks for NMI/#DF/#MCE, dual-heartbeat NMI watchdog, IBPB/L1D_FLUSH/VERW speculative-execution mitigations on VM-entry, capability-gated logging, panic-time crash dump, ELF entry-in-segment validation, ext2 metadata-block free guards, and dozens more line items. A 219-finding pre-production audit was closed across 60 fix commits; subsequent whole-system and Phase 9 audits added another ~300 findings, also closed. See [docs/HARDENING_INVENTORY.md](docs/HARDENING_INVENTORY.md) for the complete state.

---

## Performance

Measured on bare-metal Intel i3-7100U at 2.4 GHz:

| Operation                       | Latency   | Notes                            |
|---------------------------------|-----------|----------------------------------|
| `pmm_alloc_page`                | 52 ns     | Per-CPU page cache hit           |
| `pmm_free_page`                 | 51 ns     | Per-CPU page cache hit           |
| `pmm_alloc_pages(512)` (2 MB)   | 45 ns     | Buddy bulk path                  |
| `kmalloc(64)`                   | 41 ns     | Slab magazine hit                |
| `kfree(64)`                     | 39 ns     | Slab magazine hit                |
| `kmap` 4 KB round-trip          | 280 ns    | Map + unmap + TLB invalidate     |
| TLB shootdown (4 CPUs)          | ~1850 ns  | IPI to 3 APs + ack               |
| Shared-memory load              | ~5 ns     | L1 through EPT/NPT               |
| VM exit round-trip (hypercall)  | ~161 ns   | Bare metal, capability-checked   |

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

# QEMU with KVM (recommended for development)
make run-kvm

# QEMU TCG (no virtualization required)
make run

# Automated test suite (exits 0 on baseline match, halts on deviation)
make test
```

Bare metal:

```bash
# Write the ISO to a USB drive
sudo dd if=veridian.iso of=/dev/sdX bs=4M status=progress

# Boot from USB. Disable Secure Boot if needed.
# Tests run automatically; the shell launches on success.
```

### Boot modes

GRUB presents three entries:

| Entry             | Default | Description                                                            |
|-------------------|---------|------------------------------------------------------------------------|
| Safe Mode         | Yes     | Block-device writes disabled. Safe for machines with real data.        |
| Debug Mode        |         | Enables `verbose=1` — full init output on screen and serial.           |
| Storage Test Mode |         | Unlocks block-device writes. Only for dedicated test disks.            |

---

## What It Does

### Boot sequence

1. GRUB loads the kernel via Multiboot2 (legacy BIOS or UEFI).
2. 32-bit to 64-bit transition; 4 GB identity-mapped initial page tables.
3. Serial, VGA, and framebuffer console initialization, including above-4 GB UEFI framebuffers via PAT + write-combining mapping.
4. Command-line parsing (`safe_mode`, `disk_write`, `verbose`, `fb=off`, `nosmt`, `iommu`, others).
5. Buddy PMM init (3 zones DMA/DMA32/NORMAL, per-CPU page caches, scales to 1 TB).
6. IDT with 32 exception handlers and per-CPU IST stacks.
7. Kernel page tables with W^X enforcement and IPI-based TLB shootdown.
8. SMP startup — every core online via INIT/SIPI, ACPI MADT topology.
9. Slab allocator with 16 size classes and per-CPU magazines.
10. IOAPIC + IRQ vector allocation + LAPIC timer.
11. PCI/PCIe enumeration with ECAM, bridge recursion, MSI/MSI-X capability walking.
12. IOMMU (Intel VT-d or AMD-Vi) initialization with boot-time device auto-attach + interrupt remapping.
13. Storage drivers: AHCI (SATA), NVMe, virtio-blk — all interrupt-driven.
14. USB drivers: xHCI (USB 3.0) with babble + transaction-error recovery, EHCI (USB 2.0); USB-MSC for mass storage; USB-HID for keyboard and mouse.
15. Filesystem mount with foreign-FS detection; ext2 with block cache, transactions, snapshot versioning.
16. Hardening activation: SMEP/SMAP, stack guards, NMI watchdog, spec-mit MSRs.
17. Hypervisor initialization (VMX or SVM, auto-detected by CPUID).
18. AP scheduler activation — every CPU enters its own scheduler loop.
19. Automated kernel-side test suite inside an isolated domain, with per-platform baseline verification.
20. Boot summary, then the interactive busybox-on-ext2 shell.

### Domain lifecycle

```
CREATE  ->  Allocate physical memory (scatter-mapped, demand-paged)
        ->  Build EPT/NPT (GPA -> HPA)
        ->  Initialize VMCS (Intel) or VMCB (AMD)
        ->  Grant capabilities (MEMORY, IO, IPC, DOMAIN, NETWORK, ...)

RUN     ->  VMLAUNCH/VMRUN enters the domain
        ->  EPT/NPT enforces memory boundaries
        ->  Timer fires -> VM exit -> preempt if quantum expired
        ->  Hypercall  -> capability check -> execute or deny -> resume
        ->  Syscall    -> Linux-compatible handler -> resume

FORK    ->  Copy-on-write EPT/NPT fan-out via EPT root copy
        ->  Parent and child share physical pages until first write
        ->  Each write faults, splits the page, patches the writer's EPT

DESTROY ->  Roll back any active transactions
        ->  Free VMCS/VMCB, page tables, physical memory
        ->  Reparent any orphaned children
```

### Transaction flow

```
txn_begin   ->  Clear write bit on every domain page in EPT/NPT
            ->  Flush TLB

write fault ->  Copy original page to undo log
            ->  Mark page writable, resume

txn_commit  ->  Discard undo log (O(1))
txn_abort   ->  Restore pages from undo log (O(n modified pages))
```

### Security model

Every privileged hypercall passes through capability checks. Domains hold typed access tokens in a table that lives in hypervisor memory. Domains cannot forge, modify, or escalate capabilities — the table is not in any domain's address space.

```
Domain A                      Hypervisor                    Domain B
   |                              |                             |
   |-- VMMCALL(SPAWN) ----------->|                             |
   |                              |-- check DOMAIN+EXEC cap     |
   |                              |-- domain_create()           |
   |<-- domain ID ----------------|                             |
   |                              |                             |
   |-- read GPA 0x2000000 ------->|                             |
   |                              |-- EPT: NOT MAPPED           |
   |<-- #EPT violation, killed ---|                             |
```

See [docs/SECURITY_MODEL.md](docs/SECURITY_MODEL.md) for the full threat model and the per-mitigation status table.

---

## Linux as a Domain

A Linux guest is just a domain whose memory contains a Linux kernel image and whose configuration includes synthetic ACPI tables and a multi-vCPU layout. The same `domain_create()` path that launches a 16 MB native domain launches a 2 GB Linux guest. The same EPT/NPT, capability system, and scheduler apply.

Two flavors plus a hybrid:

- **Flavor A (virtio-only):** the guest sees synthetic devices (virtio-net, virtio-blk, virtio-gpu, virtio-console). The hypervisor owns the real hardware. No IOMMU is required; isolation is already provided by EPT/NPT and the absence of any guest path to real hardware.
- **Flavor B (PCIe passthrough):** the guest gets exclusive ownership of one or more real PCIe devices. The guest's Linux drivers (`i915`, `iwlwifi`, `nvme.ko`, etc.) drive the silicon directly. IOMMU isolation is mandatory and is provided by VT-d / AMD-Vi.
- **Hybrid:** within a single Linux guest, mix passthrough devices (typically GPU, USB) with virtio devices (typically storage, network). Or across guests on the same host, use Flavor A for tenant workloads and Flavor B for workstation guests that need real hardware.

This is what makes "every Linux driver is yours for free" true — the hypervisor never has to write a GPU, WiFi, audio, or Bluetooth driver, because Linux already has one and IOMMU passthrough lets the guest use it directly. See [docs/DOMAIN_ARCHITECTURE.md](docs/DOMAIN_ARCHITECTURE.md) for the deployment patterns and IPC layers.

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
    write <name> <data>     Write data to file
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

  Transactions + versioning
    txn <sub>               Transaction control (begin/commit/abort)
    versions <file>         List file versions
    revert <file> <ver>     Roll file back to version

  Execution
    exec <file>             Load + run an ELF binary from ext2

  Networking
    net                     Network device info
    ping [ip]               Ping (default: 10.0.2.2)

  Test / bench
    test isolation          EPT/NPT memory isolation test
    test txn                Transaction commit/rollback test
    bench                   VM exit latency benchmark
    bench pmm               PMM allocator benchmark
    bench kmalloc-64        Heap 64B hot path
    bench kmap-4k           kmap 4KB MMIO latency
    bench tlb-flush-1       TLB shootdown 1 page

  Diagnostics
    pci                     List PCI devices
    disk                    Disk info + read sector 0
    ahcidebug               AHCI state dump
    nvmedebug               NVMe state dump
    iommu                   IOMMU status (per-vendor)
    mouse                   USB mouse event test
    xray <sub>              Diagnostic dump
                            (pmm|pci|cpu|ahci|nvme|vfs|domain|
                             blk|kmap|heap|irq|sched|iommu|all)
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
    injectfail [sub]        Inject fault into block / driver stack
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
│   ├── domain.c                   Domain lifecycle, demand paging, COW fork
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
│   ├── smp.c                      SMP boot, AP entry, ticket locks, x2APIC
│   ├── acpi.c                     RSDP / MADT / DMAR / IVRS parsing
│   ├── pci.c                      PCI/PCIe enumeration, ECAM, MSI/MSI-X
│   ├── iommu.c                    Common IOMMU layer
│   ├── iommu_intel.c              Intel VT-d (DMA + interrupt remapping)
│   ├── iommu_amd.c                AMD-Vi (DMA + interrupt remapping)
│   ├── ahci.c                     SATA driver (interrupt-driven)
│   ├── nvme.c                     NVMe driver (with AMD MSI-timeout downgrade)
│   ├── virtio_blk.c               virtio block device
│   ├── virtio_net.c               virtio NIC (MSI-X + soft-IRQ RX)
│   ├── usb/xhci.c / usb/ehci.c    USB 3.0 / 2.0 host controllers
│   ├── usb/usb_msc.c              USB mass-storage class driver
│   ├── usb/usb_mouse.c            USB HID mouse driver
│   ├── irq.c / ioapic.c           IRQ vector allocator, IOAPIC driver
│   ├── softirq.c                  Soft-IRQ infrastructure (network RX)
│   ├── framebuffer.c              UEFI pixel console + PAT/WC mapping
│   ├── vga.c / serial.c           VGA text + COM1 serial console
│   ├── vfs.c                      VFS layer
│   ├── ext2.c                     ext2 filesystem (read + write + versions)
│   ├── bcache.c                   Block cache
│   ├── blk_dev.c                  Block-device abstraction
│   ├── blk_recovery.c             Driver-level recovery + fault injection
│   ├── gpt.c                      GPT partition table parser
│   ├── net_port.c                 Network port + timer integration
│   ├── assert.c                   VERIDIAN_ASSERT runtime
│   ├── fuzz.c                     HCALL + syscall fuzzer
│   ├── stress.c                   Slab + PMM stress harness
│   ├── fault_inject.c             Fault-injection framework
│   ├── stack_protect.c            Stack canary support
│   ├── stack_guard.c              Per-CPU stack guard pages
│   ├── watchdog.c                 NMI watchdog (tick + sched heartbeats)
│   ├── crashdump.c                Panic capture + serialization
│   ├── log_ring.c                 dmesg ring buffer
│   ├── cpu_features.c             CPUID, PAT, spec-mit MSRs
│   ├── cmdline.c                  Boot command line parser
│   ├── timer.c                    LAPIC timer + PIT fallback
│   ├── idt.c / idt_asm.asm        Interrupt descriptor table + stubs
│   ├── ap_trampoline.asm          AP SIPI entry
│   ├── test_safety.c              Kernel-side safety tests + per-platform baselines
│   └── xray.c                     Runtime diagnostics (12+ subsystems)
├── net/
│   ├── core/                      Net library: pbuf, eth, arp, ip, icmp,
│   │                              udp, tcp, dhcp, dns, sockets, loopback
│   └── test/                      Network-stack regression tests
├── domains/
│   ├── common/                    Shared headers (hypercall.h, syscall.h)
│   ├── init/                      Init domain
│   ├── test/                      Test domain
│   └── shell/                     Interactive shell
├── userland/
│   ├── musl-patch/                Port of musl libc to VeridianOS syscalls
│   ├── busybox/                   Busybox build config + integration
│   └── tests/                     Userland test binaries (incl. net_smoke)
├── lib/
│   ├── string.c                   memcpy, memset, strcmp, strlen
│   └── printf.c                   kprintf family
├── include/
│   ├── types.h                    Base types, bool, PACKED, ALIGNED
│   ├── config.h                   Build-mode flags (debug vs release)
│   ├── x86.h                      Port I/O, MSRs, CPUID, CR registers
│   ├── vmx_defs.h / svm_defs.h    VMCS / VMCB layouts, exit codes, hypercalls
│   ├── iommu_defs.h               Common IOMMU definitions
│   └── multiboot2.h               Multiboot2 structures
├── Makefile                       Build system (-O2, -Werror, -mno-sse)
├── linker.ld                      Kernel at 1 MB physical
├── grub/grub.cfg                  GRUB menu (Safe / Debug / Storage Test)
├── docs/                          See "Documentation" below
├── CONTRIBUTING.md                Build instructions and design rules
└── LICENSE                        Apache 2.0
```

---

## Roadmap

| Phase | Status | Description                                                              |
|-------|--------|--------------------------------------------------------------------------|
| 0     | Done   | Boot safety, block-device write protection, foreign-FS detection         |
| 1     | Done   | Buddy PMM, kernel page tables (kmap/W^X/TLB shootdown), slab allocator   |
| 2     | Done   | IOAPIC, MSI/MSI-X, interrupt-driven NVMe + AHCI                          |
| 3     | Done   | Preemptive scheduler, per-CPU run queues, SMP, ticket locks              |
| 4     | Done   | Full PCI/PCIe enumeration, ECAM, bridge recursion, BAR management        |
| 5     | Done   | Variable domain memory, demand paging, shared memory IPC                 |
| 6     | Done   | ext2 filesystem, block cache, FS transactions, snapshot versioning       |
| 7     | Done   | Syscall ABI, musl libc, fork/exec, busybox shell, record/replay          |
| 8     | Done   | Hardening (canaries, SMEP/SMAP, guard pages, watchdog, crash dump)       |
| 9     | Done   | Network stack: ARP, IPv4/v6, ICMP, UDP, TCP+SACK+DSACK, DHCP, DNS, sockets |
| 9.5   | Done   | IOMMU: Intel VT-d + AMD-Vi DMA remapping + interrupt remapping           |
| 10    | Next   | Identity + users (capability bundles, login, time-limited caps)          |

See [docs/ROADMAP.md](docs/ROADMAP.md) for the full plan through 3.0 GA, including time-travel debugging productization, attested execution, log-shipping live migration, and Linux-as-a-domain.

---

## Documentation

The project ships substantial design and engineering documentation under `docs/`:

- [docs/DOMAIN_ARCHITECTURE.md](docs/DOMAIN_ARCHITECTURE.md) — native vs Linux-guest domains, hybrid composition, IPC layers
- [docs/SECURITY_MODEL.md](docs/SECURITY_MODEL.md) — formal threat model, mitigation status, in-scope/out-of-scope
- [docs/HARDENING_INVENTORY.md](docs/HARDENING_INVENTORY.md) — comprehensive catalog of system + hardware hardening
- [docs/BARE_METAL_MATRIX.md](docs/BARE_METAL_MATRIX.md) — per-platform support matrix with verification status
- [docs/AUDIT_WAVE_MANIFEST.md](docs/AUDIT_WAVE_MANIFEST.md) — every audit finding mapped to its fix commit
- [docs/ROADMAP.md](docs/ROADMAP.md) — long-term development plan
- [docs/AMD_NVME_DIAGNOSIS.md](docs/AMD_NVME_DIAGNOSIS.md) — AMD ROG NVMe driver wedge root-cause writeup
- [docs/AMD_USB_FIX.md](docs/AMD_USB_FIX.md) — AMD ROG xHCI babble + multi-controller fix writeup

---

## Research Context

VeridianOS combines ideas from several areas of systems research:

- **Microkernel isolation** (seL4, L4) — domains communicate via IPC, not shared kernel state
- **Hardware isolation** (Xen, KVM, AWS Nitro) — CPU virtualization as the component boundary
- **Transactional memory** (Intel TSX) — applied at the OS page level via EPT/NPT write-protection
- **Record / replay** (Mozilla rr, VMware VM replay) — native in the hypervisor; foundation for production time-travel debugging
- **Preemptive multikernel** (Barrelfish) — per-CPU run queues with IPI-based coordination
- **Capability systems** (KeyKOS, EROS, seL4) — every privileged hypercall gated by unforgeable tokens
- **Linux-ABI compatibility** (gVisor, Firecracker) — ~50 syscalls, enough to run musl + busybox

The contribution is unifying EPT/NPT-based isolation with transactions, demand paging, preemptive per-CPU scheduling, a Linux syscall ABI, deterministic record/replay, IOMMU-enforced device isolation, and Linux-as-a-domain hosting in a single architecture where the same hardware mechanism provides isolation, enables transactions, and underpins scheduling.

---

## License

Apache License 2.0. See [LICENSE](LICENSE).

## Author

Ameya Borkar

---

*Generated 2026-05-16 from GitHub API. Source of truth: https://github.com/AmeyaBorkar/VERIDIAN*
