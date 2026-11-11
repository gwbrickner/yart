# Platforms and portability

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| P1 | Is 32-bit supported? | 64-bit only · 64-bit shipped but code kept word-size clean · 32-bit ports too | Ship 64-bit only, but use fixed-width and pointer-sized types everywhere so a 32-bit port stays possible. | Claude's pick, approved |
| P2 | Is big-endian supported? | Little-endian only · Endian-clean throughout | Little-endian only. Disk and network formats still go through explicit byte-order helpers. | Claude's pick, approved |
| P3 | What RISC-V baseline does the kernel need? | RV64GC · RV64IMAC · RV64IMA plus the privileged spec | Kernel needs only RV64IMA plus the privileged spec, with floating point left to userland. That keeps the later CPU's job small. | Claude's pick, approved |
| P4 | What x86\_64 baseline? | Any x86\_64 · x86-64-v2 · x86-64-v3 (AVX2) | Any x86\_64. No kernel code depends on newer extensions. | Claude's pick, approved |
| P5 | Which x86 firmware? | UEFI only · UEFI and legacy BIOS | UEFI only. | Claude's pick, approved |
| P6 | How does the RISC-V kernel reach firmware? | Supervisor mode on OpenSBI · Own machine-mode code | Our own machine-mode firmware. It implements the SBI calls the kernel needs, so the kernel still runs in supervisor mode and stays compatible with standard firmware on real boards. | Graeson, detailed by Claude |
| P7 | How is hardware discovered? | ACPI on x86, device tree elsewhere · Hard-coded per board | ACPI on x86, device tree on RISC-V and ARM, both feeding one internal device list. | Claude's pick, approved |
| P8 | What is the contract between `kernel/` and `arch/`? | Fixed set of headers each port implements · Function-pointer tables · Mix | Function-pointer tables for the whole contract. Each port registers its ops tables (CPU, MMU, interrupts, timer, context switch) at boot. | Graeson |
| P9 | What page size? | 4 KiB fixed · Configurable per port | 4 KiB everywhere, always through a `PAGE_SIZE` constant. | Claude's pick, approved |
| P10 | When does multicore arrive? | Phase 1 · Designed in early, switched on in phase 4 · Never | Locks and per-CPU data are multicore-safe from phase 1. The second core turns on in phase 4. | Claude's pick, approved |
| P11 | What is the minimum RAM? | 64 MiB · 256 MiB · 1 GiB | Shell in 64 MiB, desktop in 256 MiB. Small targets keep an FPGA board in range later. | Claude's pick, approved |
| P12 | Which QEMU machines are the reference? | x86 `q35` or `pc` · RISC-V `virt` · ARM `virt` | `q35` with UEFI firmware, RISC-V `virt`, ARM `virt`. | Claude's pick, approved |
| P13 | Which ports must always pass CI? | x86\_64 only · x86\_64 and RISC-V · All three | x86\_64 and RISC-V from day one, AArch64 added once phase 4 lands. | Claude's pick, approved |
| P14 | Any RISC-V or ARM boards? | None · Raspberry Pi · A RISC-V single-board computer | None before phase 6. Decide then, based on what has good documentation. | Claude's pick, approved |
