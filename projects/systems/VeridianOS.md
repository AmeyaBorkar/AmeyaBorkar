# VeridianOS

> A bare-metal x86-64 **type-1 hypervisor microkernel** that uses CPU virtualization extensions (Intel VT-x/EPT, AMD-V/NPT) as its primary OS isolation mechanism. Every OS component — the test suite, the shell, busybox, the init demo, spawned children — runs as a hardware-isolated *domain* inside its own nested page-table tree. The hypervisor boots from Multiboot2/GRUB into 64-bit long mode, brings up SMP, a buddy PMM + slab heap, AHCI/NVMe/virtio/USB block drivers, an ext2 filesystem, an IOMMU (AMD-Vi / Intel VT-d), and a from-scratch TCP/IP stack, then runs domains under VMX/SVM with a capability-gated hypercall + Linux-compatible syscall ABI.

| Field | Value |
|---|---|
| Repository | https://github.com/AmeyaBorkar/VERIDIAN |
| Visibility | Private |
| Category | systems |
| Primary language(s) | C (freestanding, C11) + x86-64 assembly (NASM) |
| Local path | `C:\Users\ameya\Documents\OperatingSystem` |
| Default branch | `master` (the local clone's checked-out branch; not `main`) |
| Lines of code (computed) | **70,788** lines across `.c`/`.h`/`.asm`/`.ld` (71,046 including `.sh`). Breakdown: C 59,395 / headers 10,241 / NASM 1,024 / linker scripts 128 / shell 258 |
| Source files (computed) | **203** code files (`.c` 102, `.h` 91, `.asm` 6, `.ld` 4); 247 total tracked files including docs/build scripts |
| Key components | VMX/SVM monitor, EPT/NPT, domain/scheduler, capabilities, COW transactions, ext2+VFS+FS journal, IOMMU (dual-vendor), TCP/IP stack, USB (xHCI/EHCI), record/replay, ~160-test safety harness |
| License | MIT-style (10.7 KB `LICENSE`) |
| Last commit | `71fc953` — "xhci: HID Report Descriptor parser — decode device's actual report layout" (2026-05-30) |

---

## What it actually is

VeridianOS is a single statically-linked freestanding ELF kernel (`veridian.bin`, linked at 1 MB) that acts as a **type-1 hypervisor**: it owns the bare metal, then runs ordinary programs not as ring-3 processes under a shared kernel but as **guests under VMX (Intel) or SVM (AMD)**, each in its own EPT/NPT nested page-table tree. The central architectural bet — and it is reflected faithfully in the code — is that the CPU's second-level address translation *is* the process isolation boundary. A domain literally cannot name another domain's host physical memory; the MMU does not translate the GPA. There is no shared syscall kernel surface reachable from a domain except through `VMCALL`/`VMMCALL` hypercalls, every one of which passes through a capability check.

The boot path in `hypervisor/main.c` (`kernel_main`, ~733 lines) is the best map of what really runs. After validating the Multiboot2 magic it brings up, in order: serial/VGA/framebuffer consoles, ACPI RSDP discovery, cmdline parsing (the `safe_mode`/`disk_write` safety gate), buddy PMM (`pmm_init` + self-test), IDT, IA32_PAT (write-combining), dynamic kernel page tables (`kmap_init`), log ring + crash dump, stack guard pages, SMEP/SMAP, spectre/MDS mitigations (`cpu_spec_mit_init`), ACPI MADT parse, SMP startup, slab `kmalloc`, IRQ/IOAPIC infra, the NMI watchdog, the LAPIC timer, the **IOMMU** (`iommu_init` + per-device attach + interrupt remapping), PCI enumeration, block drivers (AHCI → NVMe → virtio-blk), soft-IRQ + net port init, GPT partition probing, USB (xHCI → EHCI fallback), then `vfs_init`/ext2 mount, hypervisor enablement (`vmx_init` or `svm_init` based on `hv_detect_vendor`), AP scheduler hand-off, the in-guest **test-suite domain**, network DHCP bring-up, and finally an interactive **shell** — preferring `/bin/sh` (busybox) loaded as an ELF from ext2, falling back to the built-in shell domain.

The codebase is large (~71 k LOC) and unusually heavily commented: nearly every function carries an audit tag (e.g. `CRIT-006`, `SCHED-005`, `TXN-A04`, `HV-001`) describing a specific bug that was found and fixed, often with the bare-metal machine and date it was reproduced on. This is real engineering history, not boilerplate — the comments name concrete race windows, deadlock orderings, and hardware quirks (e.g. the Kaby Lake i3-7100U framebuffer-MMIO-stall that wedges the watchdog, leading to the `fb=off` cmdline escape hatch).

It targets and is claimed to boot on real hardware across both vendors (Intel Kaby Lake / Tiger Lake, AMD Zen 5 ROG) plus QEMU TCG and KVM. The development model is phase-based ("Phase 5a", "Phase 7d", "Phase 9.5"), with a per-boot-mode test baseline that halts the boot on any deviation.

---

## Architecture & how it's structured

Layering, bottom to top (each layer is code that actually exists in-tree):

```
boot/boot.asm, boot/multiboot2.asm   32-bit MB2 entry → identity-map 4 GB → long mode → kernel_main
linker.ld                            ENTRY(_start); .=1M; W^X-friendly 4K-aligned .text/.rodata/.data/.bss
│
├─ lib/            string.c, printf.c          freestanding libc subset (no glibc in kernel)
├─ hypervisor/pmm.c, kmap.c, kmalloc.c         buddy PMM (DMA32/NORMAL zones), dynamic page tables, slab heap
├─ hypervisor/smp.c, acpi.c, ioapic.c, irq.c   MADT parse, AP trampoline (ap_trampoline.asm), IDT, MSI/MSI-X
├─ hypervisor/idt.c, timer.c, watchdog.c       exceptions, LAPIC timer (~10 ms), NMI heartbeat watchdog
│
├─ Hardware hypervisor:
│   vmx.c (3189 LOC) + vmx_asm.asm            Intel: VMXON, VMCS setup, vmx_launch_guest, exit dispatch
│   svm.c (2765 LOC) + svm_asm.asm            AMD:   VMRUN, VMCB setup, svm_launch_guest, exit dispatch
│   ept.c (340 LOC)                           shared EPT/NPT 4-level walk/map/unmap/destroy (bit-compatible)
│
├─ Domain & execution model:
│   domain.c (2394 LOC)                        domain struct, create/destroy/fork, COW, demand-paging, SHM IPC
│   sched.c (394 LOC)                          per-CPU round-robin run queues, ticket locks, block/wake
│   capability.c (196 LOC)                     typed unforgeable caps, ABA-protected handles
│   txn.c (415 LOC)                            EPT/NPT write-protect + COW undo-log transactions
│   syscall.c (4052 LOC)                       ~70 Linux syscalls (file/proc/signal/socket/mmap)
│   replay.c (365 LOC)                         deterministic record/replay (compile-gated)
│
├─ Storage:
│   pci.c, ahci.c, nvme.c, virtio_blk.c        block device backends
│   blk_dev.c, blk_recovery.c, bcache.c        block layer, hot-plug recovery, block cache
│   gpt.c, ext2.c (2505 LOC), vfs.c, fstxn.c   GPT, ext2 (read+write+snapshots), VFS, FS journal
│
├─ USB:  usb/xhci.c (2612 LOC), ehci.c, usb_msc.c, usb_mouse.c
├─ IOMMU: iommu.c (dispatch) + iommu_amd.c (828) + iommu_intel.c (709)
│
├─ Network (net/): portable TCP/IP that also links into a host test harness
│   net/core/: ethernet, arp, ip4/ip6, icmp4/icmp6, udp, tcp (SACK/DSACK/Reno), socket, dhcp, dns, loopback
│   hypervisor/net_port.c, net_nic_virtio.c, virtio_net.c, softirq.c   kernel glue + virtio-net MSI-X RX
│
├─ Hardening: stack_protect.c (canaries), stack_guard.c (guard pages), cpu_features.c (SMEP/SMAP/IBRS),
│             watchdog.c, crashdump.c, log_ring.c, assert.c, fault_inject.c, fuzz.c, stress.c
│
└─ domains/ (separate freestanding binaries, embedded as raw bytes via `ld -r -b binary`):
    init/domain_init.c, shell/shell.c (1268 LOC), test/test_suite.c (681 LOC)
   userland/ : musl + busybox build scripts, syscall_arch.h port, C test programs
```

**Control flow at runtime:** `sched_cpu_loop` (each CPU) → `sched_pick_next_cpu` → `vmx_dispatch_load`/`fxrstor` → `vmx_launch_guest`/`svm_launch_guest` (asm) → guest runs → VM-exit → `vmx_exit_handler`/`svm_exit_handler` dispatch on exit reason → `VMCALL`→ hypercall/syscall handler → resume. Timer ticks fire in the LAPIC ISR (`sched_timer_tick`), decrement the running domain's quantum atomically, and signal preemption.

**Two ABIs into the hypervisor**, both over `VMCALL`/`VMMCALL`:
- **Native hypercalls** (`HCALL_*`, defined in `include/vmx_defs.h`, 0x01–0xFF) — VeridianOS-specific: txn begin/commit/abort, spawn/kill/ps, xray diagnostics, disk/fs/net operations, fuzz/stress/inject, test-result reporting.
- **Linux syscalls** (`SYS_*`, base offset `SYSCALL_BASE`) — RAX≥SYSCALL_BASE routes into `syscall.c`'s dispatch, letting unmodified musl/busybox binaries run.

---

## Code walkthrough (the important parts)

### Boot & long-mode entry (`boot/boot.asm`)
Pure 32-bit→64-bit bring-up: checks CPUID + long-mode support, builds a static identity-mapped 4 GB page-table tree (PML4→PDPT→4×PD of 2 MB pages, deliberately covering the LAPIC at 0xFEE00000 near 4 GB), enables PAE + LME + paging, loads a 64-bit GDT, far-jumps to long mode, zeroes BSS (from `__bss_zero_start`, *skipping* the boot page tables), reloads saved Multiboot2 `eax`/`ebx`, and calls `kernel_main`. The BSP stack is a fixed 16 KB with no guard page during early boot (comment `BOOT-007` notes the worst-case measured depth is ~6 KB).

### EPT/NPT (`hypervisor/ept.c`)
A single 4-level page-table implementation serves *both* Intel EPT and AMD NPT, because the relevant bits are identical (present=bit0, write=bit1, large=bit7, same address mask). `txn.c` even enforces this with `_Static_assert`s. Key functions: `ept_create` (allocates the PML4 from the **DMA32 zone** so phys==virt identity casts are valid), `ept_walk_alloc`/`ept_walk_alloc_2mb`, `ept_map_page`/`ept_map_2mb`/`ept_map_range` (auto-promotes to 2 MB pages when aligned unless `EPT_NO_LARGE_PAGE` is set — domain memory uses 4 KB granularity so the COW/txn write-fault path works), `ept_gpa_to_hva` (host-side translation through the domain's table), `ept_invept_single`, and `ept_destroy_with_data` which walks the whole tree freeing leaf data pages while honoring `EPT_SHM_BIT` (skip shared pages) and `EPT_COW_BIT` (decrement refcount, free only at zero). Software-available PTE bits carry the COW/SHM markers.

### Domains, COW fork & demand paging (`hypervisor/domain.c`)
The `struct domain` (in `domain.h`) is carefully laid out — the first ~104 bytes have **hardcoded offsets that assembly depends on** (id, state, vmcb/vmcs phys, npt/ept roots, mem base/size, entry/stack, GPRs). It also carries the cap table, per-domain txn state + lock, COW lock, fd table (64 entries) + lock, cwd, signal state (handlers/masks/restorers per signal), brk/mmap allocators, FS/GS base, and a 16-byte-aligned 512-byte `fxsave_buf`.

- `domain_create` / `domain_create_trusted`: trusted domains (shell/test/init) additionally get `CAP_TYPE_DEBUG` + a **wildcard `CAP_TYPE_DOMAIN_KILL`**; spawned workers do not (the `HV-001/004` hardening that removed the legacy self-cap that bypassed every check).
- `domain_fork` + `ept_clone_cow`: clone walks the parent tree, sets every leaf **read-only + COW** in *both* parent and child, and CAS-bumps a global `page_refcount[]` (uint16, saturating) — refusing large pages outright (`H-20`). `cow_handle_fault` is invoked from the EPT-violation/NPF handler: if refcount==1 the page is converted in place to writable (no copy, refcount reset); otherwise it allocates a fresh DMA32 page, `memcpy`s, maps it writable, and decrefs the old.
- `domain_demand_page`: allocates+zeroes+maps a page on a not-present GPA fault, growing `pages_allocated`.
- Shared-memory IPC: `shm_create`/`shm_grant`/`shm_revoke`/`shm_destroy` with ownership-gated lookup (`shm_info_for_caller`), regions tracked via `EPT_SHM_BIT` leaf pages, 32 regions × ≤256 pages × ≤16 mappings.

### Scheduler (`hypervisor/sched.c`)
Per-CPU round-robin run queues (`cpus[].run_queue`, ticket spinlock `rq_lock`), fixed `SCHED_QUANTUM_TICKS` quantum, no priorities. The dead global run-queue path was deleted (`SCH-003`). `sched_pick_next_cpu` first sweeps for expired `BLOCK_NANOSLEEP` timers (TSC-based via calibrated `tsc_ticks_per_us`), then advances `rq_idx` round-robin, transitioning READY→RUNNING **inside the lock** to prevent double-pick. `sched_cpu_loop` does the FXSAVE/FXRSTOR around launch (VMX/SVM hardware does not preserve x87/SSE state across VM entry/exit — this closes a cross-guest XMM/YMM information leak, `SCHED-006`). Block/wake use atomic CAS on `dom->state` with explicit acquire/release fences and a lost-wakeup recheck against `sig_pending` (`SCHED-002`).

### Capabilities (`hypervisor/capability.c`)
A fixed-size per-domain cap table of typed entries (NULL/MEMORY/IPC/IO/DOMAIN/IRQ/DEBUG/DOMAIN_KILL/DOMAIN_SPAWN/NETWORK) with R/W/X/GRANT/REVOKE perms. `cap_check` matches by type + object_id (exact, `CAP_OBJECT_WILDCARD`, or grant-side IO wildcard) + required perms. There's a per-slot `generation` counter and an ABA-protected `cap_check_handle` that rejects stale (slot,gen) pairs across revoke/recreate. `cap_grant` was deliberately **removed** (`DC-001..003`) so untrusted domains cannot propagate wildcard rights. Note: the gen/handle ABA path has **no in-tree callers yet** — it exists so the counter isn't dead code and the audit test can exercise it (explicitly stated in the comments).

### Transactions (`hypervisor/txn.c`)
Per-domain memory transactions implemented purely via nested-page-table write-protection. `txn_begin` write-protects every domain leaf (skipping SHM pages) and flushes the TLB (INVEPT / VMCB TLB-flush). On the first write to each page, `txn_handle_write_fault` copies the original into a `pmm_alloc`'d backup, records (gpa, copy_hpa) in a bounded undo log, and re-grants write. `txn_commit` is O(1) (free backups, restore write perms). `txn_abort` restores each page via `domain_memcpy_to_gpa` (which triggers COW split correctly) and frees backups — and it carefully **drops `txn_lock` during the restore loop** to avoid an A↔B deadlock with `cow_lock` on the demand-page path (`LOCK-001`). `txn_cleanup` rolls back automatically if a domain dies mid-transaction. All entry points serialize on `dom->txn_lock`; the txn state lives *inside* the domain struct (not a global table indexed by `id % MAX_DOMAINS`, which used to alias domains after id wrap — `TXR-002`).

### Syscall layer (`hypervisor/syscall.c`, 4052 LOC)
The largest single file. 76 `case SYS_*` arms over a dispatcher (70 distinct `SYS_*` defines in `domains/common/syscall.h`). Implements file I/O (open/close/read/write/lseek/stat/fstat/lstat/getdents64/mkdir/rmdir/unlink/rename/readlink/getcwd/chdir/access), process model (fork/execve/waitpid/getpid/getppid/exit/kill), pipes, signals (rt_sigaction/rt_sigprocmask/rt_sigreturn with on-guest-stack signal frames + per-signal restorer trampolines), memory (brk/mmap), TLS (arch_prctl FS/GS base), ioctl/fcntl/dup/writev/poll/nanosleep/uname, and a **full BSD socket family** (socket/bind/listen/accept/connect/sendto/recvfrom/shutdown/setsockopt/getsockopt/getsockname/getpeername) routed into `net/core/socket.c`. All guest pointers are validated through `dom_gpa_range_valid` / `dom_strnlen` before translation. There's a guard that read-only filesystem syscalls are rejected when writes aren't permitted.

### VMX / SVM monitors (`hypervisor/vmx.c`, `hypervisor/svm.c`)
`vmx_init` does the full Intel dance: checks support, configures `IA32_FEATURE_CONTROL` (VMXON-outside-SMX + lock), adjusts CR0/CR4 per the fixed-bit MSRs, allocates a DMA32 VMXON region with the revision ID, sets up GDT/TSS, executes VMXON. `vmx_setup_vmcs` programs guest/host state, EPTP, MSR bitmap, and pinned/proc-based controls. `vmx_dispatch_load` handles cross-CPU `VMPTRLD` + per-CPU host-state refresh + per-EPTP INVEPT on every dispatch (the original cross-CPU dispatch bug). `vmx_exit_handler` is a big switch over exit reasons: VMCALL (gated to ring 0), CPUID, RDTSC/RDTSCP, IO_INSTR, HLT, **EPT_VIOLATION** (routes to COW/demand-page/txn-write-fault), EPT_MISCONFIG, EXTERNAL_INT, EXCEPTION_NMI, TRIPLE_FAULT, RD/WRMSR, XSETBV, CR_ACCESS, INIT/SIPI, etc. `svm.c` mirrors this for AMD with VMCB-based state and `#NPF` handling. Both disable VMX/SVM on every AP and the BSP at halt (`MODE-001/002`) so a warm reset doesn't leave the CPU in an inconsistent virt state.

### Filesystem (`hypervisor/ext2.c`, `vfs.c`, `fstxn.c`, `bcache.c`)
A real read/write ext2 implementation: superblock + block-group parse, inode read/write with direct + single/double/triple indirect blocks, directory iteration (`ext2_readdir`), path resolution, create/unlink/mkdir/rmdir/rename, block allocation/free with bitmap accounting, and **snapshot versioning** (`ext2_snapshot_save`/`list_versions`/`revert`). `fstxn.c` is a separate **filesystem journal**: `fstxn_begin`/`save_block`/`commit`/`abort`/`recover` write old block contents to a journal region (placed after the ext2 fs in the partition) so metadata changes commit atomically and roll back on crash detected at next mount. `vfs.c` sits on top with foreign-FS detection and a safety-gated `vfs_format` (refuses to auto-format a GPT/foreign disk). The block layer (`blk_dev.c`) installs a **stub write function** when writes aren't unlocked (the `safe_mode` gate), and `blk_recovery.c` does rate-limited device-reset recovery off the timer ISR.

### IOMMU (`hypervisor/iommu.c` + `iommu_amd.c` + `iommu_intel.c`)
A vendor-neutral dispatcher discovers AMD IVRS or Intel DMAR via ACPI, then drives the per-vendor backend: program Device Table Entries (AMD) / Context Entries (Intel), enable translation, and turn on **interrupt remapping** (IRT) so MSI/MSI-X deliveries route through the IOMMU. Per-device attach happens after PCI enumeration; there's a canonical post-attach summary line for bare-metal log scraping, a fault counter, and an `iommu=off` kill-switch (added after the AMD ROG Zen 5 reproducer where IOMMU-on broke xHCI Enable-Slot and NVMe completions).

### Network stack (`net/core/`)
From-scratch, written to be **portable**: the same `net/core/*.c` files compile into the freestanding kernel (with a `<string.h>` shim under `net/include/compat/`) *and* into a host `make nettest` binary linked against libc + pthread. Protocols: Ethernet, ARP, IPv4, IPv6, ICMPv4/v6, UDP, and a substantial **TCP** (`tcp.c`, 1255 LOC) with retransmit, RTT estimation, Reno congestion control (cwnd/ssthresh), fast retransmit/recovery, **SACK (RFC 2018) + DSACK (RFC 2883)** negotiated via TCP option kind 4/5, and out-of-order reassembly. Plus BSD `socket.c`, DHCP client, DNS resolver, loopback. The kernel driver path is virtio-net with MSI-X and soft-IRQ RX delivery (`hypervisor/softirq.c`, `net_nic_virtio.c`).

### Record/replay (`hypervisor/replay.c`)
Compile-gated behind `VERIDIAN_REPLAY_ENABLED` (default 1). Records VM exits, syscall args/returns, CPUID results, RDTSC values, and output chars into a per-domain log; on replay it short-circuits most syscalls by injecting the recorded RAX, but *re-executes* brk/mmap/arch_prctl because they mutate memory/page-table state subsequent instructions depend on. `replay_verify` checks bit-perfect output. Overflow halts the recording domain to prevent divergence.

---

## Key algorithms / implementation details

- **Unified EPT/NPT via bit equivalence.** Rather than duplicate every page-walk for Intel vs AMD, the code relies on the fact that present/write/large/address-mask bits coincide, enforced by `_Static_assert`. One `ept.c` serves both; `hv_vendor` only selects which root (`ept_pml4_phys` vs `npt_pml4_phys`) and which TLB-flush primitive (INVEPT vs VMCB TLB-control).

- **COW transactions = MMU write-protect + lazy undo log.** No instruction emulation; the hardware page-fault on a write-protected page is the trigger. Commit is O(1); abort is O(modified pages). Backups are real physical pages. The same `EPT_COW_BIT`/`EPT_SHM_BIT` software bits coordinate fork-COW, txn-COW, SHM, and the destroy-time free logic so they never double-free or leak shared pages.

- **Saturating refcounted COW with CAS.** `page_refcount[pfn]` is a global uint16 array; fork does a CAS loop `0→2` (first share) / `N→N+1` (saturating at UINT16_MAX). The width was deliberately widened from uint8 (`CRIT-015`: a uint8 CAS over a uint16 array livelocked/wrapped past 255).

- **Cross-guest FPU isolation.** Because VMX/SVM don't save x87/SSE/AVX across VM transitions, the scheduler round-trips each guest's FPU through its own 16-byte-aligned `fxsave_buf` on every launch iteration — otherwise guest B could read guest A's XMM secrets.

- **Lock-ordering discipline.** Documented orderings (`rq_lock` → domain fields → `ipi_lock`; `cow_lock` outer / `txn_lock` inner) plus deliberate lock-drops (`txn_abort` releases `txn_lock` during restore) to avoid A↔B deadlocks introduced once SMP preemption landed.

- **Capability ABA protection.** Per-slot generation counter bumped on revoke so a recycled slot can't satisfy a stale handle — a real defense the audit added even though no caller wires it yet.

- **Per-boot-mode test baselines.** Instead of "0 failures", the kernel matches an exact `(pass, skip, fail)` triple against a `baselines[]` table keyed on boot mode (QEMU-safe / storage-test / KVM / bare-metal / release), so silent SKIP regressions are caught. Boot halts on any deviation.

- **Spectre/MDS + W^X + SMEP/SMAP + guard pages + canaries + NMI watchdog** are all present as real init paths, not aspirational — `cpu_spec_mit_init` builds a runtime mitigation flag byte the VMX/SVM asm consumes before VMENTER (IBRS/IBPB/L1D_FLUSH/MD_CLEAR).

---

## Tech stack & build system

From the `Makefile` (toolchain: `gcc`, `nasm`, `ld`, `objcopy`):

- **Kernel CFLAGS:** `-ffreestanding -nostdlib -nostdinc -mno-red-zone -mcmodel=kernel -fno-pic -fstack-protector-strong -mstack-protector-guard=global -fno-exceptions -mno-sse -mno-sse2 -mno-mmx -mno-avx -Wall -Wextra -Werror -O2 -g`. The build SHA is baked in via `-DVERIDIAN_GIT_SHA`.
- **ASFLAGS:** `nasm -f elf64 -g -F dwarf`.
- **LDFLAGS:** `-n -nostdlib --no-undefined -T linker.ld` (the `--no-undefined` turns a missing/weak-zero symbol into a *link* error rather than a runtime NULL-call).
- **Domain CFLAGS** differ (no `-mcmodel=kernel`, `-fno-stack-protector`); each domain is compiled, linked with its own `.ld`, `objcopy`'d to a flat `.bin`, and embedded into the kernel as raw bytes via `ld -r -b binary` (referenced by `_binary_domains_*_bin_start/_end`).
- **Net library** compiles with an extra `-Inet/include/compat` so `<string.h>` resolves to the freestanding shim (kernel-side `lib/string.c` provides the symbols).
- **Release mode** (`make release` → `-DVERIDIAN_RELEASE_BUILD`) cascades through `include/config.h` to compile out asserts, fault injection, fuzz/stress hypercalls, and verbose printf, while *keeping* canaries, guard pages, SMEP/SMAP, watchdog, and crashdump.
- No external dependencies; no libc in the kernel. ISO built with `grub-mkrescue`.

---

## Build / run / test

```bash
make                       # build veridian.bin (the kernel ELF)
make iso                   # produce veridian.iso (GRUB, Safe Mode default)
make release               # clean + rebuild with VERIDIAN_RELEASE_BUILD=1

make run                   # QEMU TCG, Safe Mode, virtio disk, serial stdio
make run-kvm               # QEMU + KVM (-cpu host -smp 4), with virtio-net
make run-ext2 / run-ext2-kvm   # boot with a GPT+ext2 disk image
make run-storage-test      # DANGEROUS: boots the entry that unlocks block writes (wipes disk.img first)

make test                  # authoritative CI: TCG -smp 4 -cpu qemu64, greps for "RESULT: ✅"
make test-kvm              # faster KVM variant (skipped if /dev/kvm absent)

make nettest               # host build: net/core + net/test linked against libc+pthread, runs all suites
make bbox-applet-check     # regression guard that busybox wget/nc stay enabled in defconfig
make debug                 # QEMU with -s -S for GDB on :1234 (tools/gdb_init.gdb)

make ext2.img              # build a 64MB GPT+ext2 image (needs sudo/losetup/mkfs.ext2; installs musl+busybox)
```

`make test` boots the ISO, runs the in-guest **test-suite domain**, then the kernel's safety suite dispatches `HCALL_TEST_BASELINE_CHECK`, compares the `(pass, skip, fail)` triple to the per-mode baseline, and the Makefile fails unless it sees the `RESULT: ✅ MATCHES BASELINE` marker.

---

## Tests

Three independent test layers exist:

1. **In-guest functional suite** — `domains/test/test_suite.c` (681 LOC), ~13 tests that run *inside a domain* under VMX/SVM: identity/TSC-monotonic, **isolation probes** (own memory at GPA 0x200000 accessible; GPAs at 32/80/256 MB blocked by EPT), transaction commit/abort/multi-page, NOP-hypercall, memory R/W, deep-stack, free-memory. These exercise the actual hardware isolation boundary from the guest side.

2. **Kernel-side safety harness** — `hypervisor/test_safety.c` (**7470 LOC**, the largest file). `test_safety_baseline_check(run, fail, skip)` runs ~14 inline asserts + ~130 dispatched tests (~144–160 total depending on phase, per the `baselines[]` comments) covering boot-mode/cmdline safety (S1), block-device range checks and write-gating (S5), HCALL capability denial, VFS auto-format refusal, fault injection, SMP AP-bringup invariants, IOMMU state, log-ring wrap, etc. Results are matched against a per-boot-mode baseline triple and a boot-halting deviation check.

3. **Host network test harness** — `net/test/` (20 files): `make nettest` links `net/core/*` against `netport_host.c` (malloc + pthread mutex + virtual clock) and runs suites for pbuf, fakenic, arp, ip4/ip6, icmp4, udp, **tcp** (handshake, retransmit, close, SACK), socket, dhcp, dns, loopback, and replay. Test runners are declared `__attribute__((weak))` so partial builds still link.

Additionally `fuzz.c` (syscall fuzzer against a sacrificial domain, seed-reproducible, histogram of syscall numbers, `HCALL_FUZZ`) and `stress.c` (`HCALL_STRESS`) and `fault_inject.c` provide stress/chaos testing, all compile-gated out of release builds.

---

## Status, completeness & notable gaps

**Genuinely working (verified in code, not just claimed):** dual-vendor VMX/SVM with EPT/NPT, SMP with per-CPU scheduling, buddy PMM + slab heap, fork/exec/wait/pipes/signals/sockets syscall ABI, COW + memory transactions, ext2 read+write+snapshots + FS journal, AHCI/NVMe/virtio-blk/virtio-net/xHCI/EHCI drivers, dual-vendor IOMMU + interrupt remapping, a real TCP/IP stack with SACK/Reno, record/replay, and an extensive boot-time test harness.

**Stubs / deferred / not implemented (found in code):**
- `svm.c`: "NPT not supported. Shadow paging not implemented." — i.e. SVM **requires** NPT; there is no shadow-paging fallback.
- `iommu_intel.c`: a full second-level page table for true per-device DMA confinement is "deferred"; attach programs Context Entries but the in-tree path is closer to managed-passthrough than per-domain DMA remapping.
- `pci.c`: hot-plug is "not implemented"; some BARs reported `0xFFFFFFFF` (unimplemented/absent) are skipped.
- The capability **ABA-handle** API (`cap_handle_get`/`cap_check_handle`) has no production callers — exists for the audit test only.
- Worker-domain async dispatch to APs: the `kind` field and AP plumbing exist, but `HCALL_SPAWN` still runs synchronously on the BSP (noted as a future feature).
- ~133 `TODO`/`FIXME`/`stub`/"not implemented"/placeholder markers across the C/H tree (many are descriptive, e.g. the intentional safe-mode write *stub*).
- `kmap.c`: a TODO to unmap the AP trampoline after SMP init completes.

**Dead/legacy:** the old global run-queue scheduler path and the legacy per-domain self-cap were deleted (SCH-003 / HV-001); several `struct domain` signal fields are marked deprecated-but-kept for struct-layout compatibility.

There are also large local-only research/draft markdown files in the working tree (`REMAINING_INNOVATIONS_RESEARCH.md` 84 KB, `BUNDLE_B_RESEARCH.md`, `BUNDLE_*`/`*_RESEARCH.md`) — `.gitignore` lists patent-material patterns (`PATENT_DRAFT_INDIA.md`, `docs/Patent/`, `*_RESEARCH.md`, `BUNDLE_*`) as intentionally local-only. Binary artifacts present but not source: `veridian.bin`, `veridian.iso` (14 MB), `disk.img`, `efi_part.img`, `veridian_boot.vdi`. The `.gitignore` excludes `*.bin/*.iso/*.img/*.vdi`, but these specific blobs are nonetheless present in the working tree.

---

## README vs. code

Mostly **consistent and unusually honest** — the README's feature list matches what the code actually implements. Concrete discrepancies:

| Claim (README) | Reality (code) |
|---|---|
| "**70,398 lines** across **199 source files**" | Computed **70,788** lines (`.c/.h/.asm/.ld`; 71,046 with `.sh`) across **203** code files (102 `.c`, 91 `.h`, 6 `.asm`, 4 `.ld`). README is slightly stale/low but in the right ballpark. |
| "Linux-compatible syscall ABI (**~50 syscalls**)" | Undercount: `domains/common/syscall.h` defines **70** `SYS_*` numbers and the dispatcher in `syscall.c` has **76** `case SYS_*` arms. |
| "Capabilities are typed (MEMORY, IO, IPC, DOMAIN, NETWORK, DEBUG) … and **time-limited where appropriate**" | The capability struct has no expiry/time field; `capability.c` has no time-limit logic. "Time-limited" is not implemented. |
| "runs **hundreds** of automated tests on every boot" | The kernel safety suite is ~144–160 tests per the `baselines[]` table (low hundreds only if you also count the host nettest suites, which don't run on-boot). Reasonable but the on-boot count is ~150, not many hundreds. |
| Default branch implied `main` | The local clone is on **`master`**. |
| Architecture diagram shows a "**Linux guest** (full Linux + userland)" domain | No full-Linux-guest boot path exists in-tree; domains run native binaries or musl/busybox ELFs. This is aspirational in the diagram. |
| "shared-memory load: **~5 ns (17–27 cycles)**" / specific perf numbers | Plausible but not independently verifiable from source alone (benchmark hypercalls like `HCALL_PMM_BENCH` exist; the quoted figure isn't a code artifact). |

The README's IOMMU, TCP/SACK/Reno, record-replay, transaction, COW-fork, and demand-paging descriptions all check out against the code.

---

*Doc generated 2026-06-16 by reading the source at C:\Users\ameya\Documents\OperatingSystem. Source of truth: https://github.com/AmeyaBorkar/VERIDIAN*
