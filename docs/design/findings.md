# Findings from the design pass

Gaps and conflicts the design-note agents found while turning decisions into designs. Each one needs a fix in a note, a roadmap change, or an answer from Graeson. Items that need Graeson are also in `../open-questions.md`. Delete an item when it is resolved.

## Roadmap ordering

| # | Finding | IDs | Proposed fix |
| --- | --- | --- | --- |
| R1 | Mutexes, wait queues and deferred work need kernel threads in phase 1, but the scheduler is phase 2 | K3, K4, S3 | Add a minimal kernel-thread scheduler to phase 1; phase 2 replaces its policy |
| R2 | The RISC-V firmware needs a device tree reader in slice 0.6, and phase 1 interrupts and timers need controller addresses, but device tree and ACPI tables are listed for phase 4 | P7, H4, K5 | Move a minimal device tree reader to phase 0 and static ACPI table lookup to phase 1; phase 4 keeps only the AArch64 work |
| R3 | The EFI partition is FAT from slice 0.5, but FAT is phase 6 and F13 wants our own image tools | F4, F13, B1, D9 | Phase 0 uses QEMU's directory-backed FAT drive; a host-side FAT writer lands before the VirtualBox image in phase 5; the in-kernel FAT driver stays in phase 6 |
| R4 | yartfs arrives in phase 2, the page cache in phase 3 | M8, M7, F1 | Move the page cache for file objects to phase 2 so file I/O is not written twice |
| R5 | Modules need ELF relocation in phase 4, but boot.md defers relocation code to phase 6 | K1, B11 | Shared ELF library gains relocation in phase 4 |
| R6 | CI lint is in slice 0.1 but `./x` arrives in 0.2 | T10 | Slice 0.1 ships a plain lint script that `./x lint` later wraps |
| R7 | Randomized user layout in phase 2 needs entropy before any driver provides it | X6, H16 | Early entropy from CPU instructions or the boot path in phase 2 |
| R8 | Phase 1 has no wall-clock source | K6, B5 | Boot paths pass the boot time in `BootInfo` |

## Arch contract gaps

All against `arch-contract.md` (P8). The other notes need these and the contract lacks them.

- An op to apply a module relocation (K1).
- An op to print a `TrapFrame` for the panic register dump (K9).
- `KernelTrapOps.pageFault` must return whether the fault was resolved, so user-copy recovery can run (K14).
- A defined "start at my caller" mode for `backtrace` (K9).
- A statement that upcalls run on the interrupted thread's kernel stack, so preemption on interrupt exit works (K2).
- `ArchMmuOps` must report the user address limit, the kernel-half base, the largest leaf level, and on AArch64 which root an address uses (M13).
- No address-space identifier on `activate`, so every switch flushes the TLB. Acceptable to start; note it (M13).
- The single IPI carries no reason; the kernel keeps per-CPU pending-reason bits (P10).
- P8 and `architecture.md` name five tables; the contract has nine plus an upcall table. Update the wording.

## BootInfo gaps

Against `boot.md` (B5). Fields the other notes need: `firmwareRuntime` (UEFI system table, for reset), `bootCpuId`, `magic`, `version`, boot wall-clock time, the direct-map base, and a statement of which loader memory the kernel may free.

## Decision wording to fix

- **T7**: clang-format cannot check identifier case; clang-tidy's naming check does. Reword.
- **M13**: "same tree shape" is not literally true for Sv39 (three levels) or AArch64's split roots. The walker takes its depth from the port.
- **T5**: "one cross file per architecture", but the UEFI loader is a COFF target and the kernel an ELF one. Allow a second cross file or a per-target override.
- **T9**: address checking has no runtime on yart userland. Read it as "userland code built on the host" until we write one.
- **T12**: the assembly allow-list omits userland entry code and syscall stubs.
- **H17**: "every driver has a QEMU test" cannot hold for hardware QEMU does not model. Those get a manual checklist.

## Unsettled, needs a call

- RISC-V paging mode: Sv39 or Sv48 (P3, M13). Sv39 suits a small custom CPU.
- RISC-V userland baseline and float ABI (P3). Only the kernel's is decided.
- AArch64 baseline (P3, P4 have no equivalent).
- Whether linking the compiler's runtime library counts as third-party code (D7). The notes assume it does and supply helpers from `lib/`.
- Language of `./x` and host scripts; Python 3 proposed (T5).
- Where interrupt-controller code lives: `arch/` or `drivers/` (H4). The contract puts it behind `ArchIrqOps`.
- x86 timer calibration: an invariant counter is not baseline, and no reference clock is named (K6, P4).
- Protection features that are not baseline x86_64 (K14, M11): supervisor-access prevention and 1 GiB pages. Use when present, never require.
- Stable module ABI: none assumed; modules check a kernel build ID (K1).
- Debug shell on the user's serial console from phase 2: escape sequence and release-build default (K10, X1).
- libyart error names: POSIX names or our own (K7, D6).
- Out-of-memory victim rule: largest resident set, never init, proposed (M10).
- Whether "compatible with standard firmware" (P6) is a real requirement. Under OpenSBI our loader half would have to run in supervisor mode, and no slice plans that.
- RISC-V boot from a disk image: the firmware has no storage driver, so QEMU preloads the boot archive (B2, F13).
- Frame comparison across ports if the rasterizer uses floating point: exact, tolerance, or per-port references (Q9, W13).
- Performance tracking: per release (Q10) or in CI (B13), and the regression threshold.
- VirtualBox, VMware, Hyper-V and real-hardware boots cannot run in CI; manual checklist (D9, R8).
- CI provider and container registry (T6, Q11).

## To verify in phase 0

- QEMU `-kernel` behaviour with a non-ELF file on RISC-V `virt`, and what arrives in `a2` (slice 0.6).
- Package names, versions, target triples and flags in `build-system.md` (slices 0.1, 0.2).

## From the processes and syscalls notes

Decision conflicts that need a call:

- **S7 is too small for D6.** The signal subset omits hangup, broken pipe, alarm, window change, quit, abort and the user signals. `abort`, `alarm` and editor resize need them, so "POSIX source runs unchanged" fails with S7 as written. Proposed fix: S7 becomes the full standard signal set.
- **Native programs would need libc.** A11 picks POSIX threads only and M15 puts `malloc` in libc, which contradicts "our own programs use libyart" (D6, A1). Proposed fix: libyart owns threads and the allocator; libc wraps them.
- **Local sockets arrive too late.** S12 has them in phase 5, but phase 3 services (init, log service, elevation service) need them (U12, U17, X4). Proposed fix: local sockets move to phase 3.
- **libyart arrives too late.** Listed in phase 3, but phase 2 user programs need its syscall stubs and start-up code (A1). Proposed fix: a minimal libyart in phase 2.
- **Are fork, exec and pid-based calls part of the native API?** They exist in the kernel only for POSIX. The note marks them `layer = posix` in the syscall table (D6, S1, S7, S8).
- **Randomized layout for static programs** in phases 2 to 4 needs self-relocating position-independent executables (S10, X6). Not decided.
- **The fixed-priority class** does not win across CPUs with per-CPU queues unless woken threads are pushed, nothing caps a spinning fixed thread, and no lock has priority inheritance (S4, S6, K3).
- **S16** does not say whether the memory limit counts mapped or resident bytes, and nothing limits process count.
- **Syscall numbers** are embedded in static programs, so renumbering means rebuilding everything. Whether numbers freeze at 1.0 is undecided (A3, S10).
- **A10**: which math library, which phase, where it lives, and whether it is exempt from the style and warning rules.
- **A12**: no trace mechanism is decided, and nothing covers debugging user programs at all.
- **Shared buffers**: a client that shrinks a shared buffer can fault the compositor; sealing is not decided (S12, W2).
- **A Linux personality** needs full signals, Linux's `/proc` and `epoll`, not only a second syscall table (G7).
- No phase is given for `poll`, `/proc` or crash reports; phase 3, 3 and 2 proposed (S13, S14, S15).
- `memory.md` lists fork in phase 2; `processes.md` puts the fork call in phase 3. Read as kernel support in phase 2, the call in phase 3.

More arch contract gaps: `TrapFrame` is opaque, so the contract needs ops to push and restore a signal frame, read the user program counter and stack pointer, dump user registers, walk a user backtrace and rewind a syscall for restart. The port must also report its ELF machine number.
