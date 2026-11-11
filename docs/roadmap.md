# Roadmap

yart is built in seven phases. A phase is a list of slices; each slice is one spec, one reviewable diff, one merge. 1.0 is the end of phase 6 (D2). The custom CPU project starts after 1.0 and is not planned here.

Required ports: x86_64 and RISC-V 64 from phase 0, AArch64 from phase 4 (D3, P13).

## Phases

| Phase | Name | What gets built | Done when |
| --- | --- | --- | --- |
| 0 | Foundations | Repo skeleton, pinned toolchain, Meson and `./x` wrapper, Kconfig, our own UEFI loader, our own RISC-V machine-mode firmware, common boot-info struct, serial console, test harness, CI | Both ports boot in QEMU through our own boot code to a kernel that prints a banner over serial, and CI proves it on every push |
| 1 | Kernel core | Arch ops tables, physical allocator, page tables, kernel heap, interrupts, one-shot timers, locks, logging, panic backtraces, debug shell | The same kernel test suite passes on both ports |
| 2 | Processes and storage | Scheduler, user mode, generated syscall table, ELF loader, VFS, tmpfs, devfs, virtio block and console, module interface, yartfs v1 with host tools, users and groups | A program stored on a yartfs disk loads and runs in user mode on both ports |
| 3 | Command line | Native system library, POSIX layer, init, TTY and pseudo-terminals, signals and job control, shell with structured pipelines, about thirty utilities, editor, manual pages, syscall tracer, framebuffer console | You can live in the terminal for an hour without missing something basic |
| 4 | Portability proof | AArch64 port, multicore on every port, runtime module loading, device tree and ACPI static tables | AArch64 reaches the shell with almost no changes outside `arch/` and `boot/` |
| 5 | GUI | Shared memory, local sockets, event queue, dynamic linking, compositor, vector rasterizer, fonts, toolkit, design system, desktop shell, login, four core apps, service manager, sandbox call, and the drivers VirtualBox needs (AHCI, PS/2, firmware framebuffer) | A usable desktop on two ports in QEMU, and x86_64 boots to the desktop in VirtualBox |
| 6 | 1.0 | Networking, USB, audio, NVMe, ACPI interpreter, real hardware, swap, FAT, yartfs v2, package manager, first ports, VMware and Hyper-V, installer, developer kit | The 1.0 checklist below is complete |

## Phase 0 slices

Specs live in `docs/specs/phase-0/`. Order matters; each depends on the ones before it.

| Slice | Name | Result |
| --- | --- | --- |
| 0.1 | Repo skeleton and toolchain | Directory layout, pinned container image, formatting and lint config, CI that runs lint |
| 0.2 | Build system | Meson cross files per port, `./x` wrapper, Kconfig wired to a generated config header |
| 0.3 | Host test harness | Unit test runner for code compiled on the host, with sanitizers |
| 0.4 | Kernel entry and boot info | Boot-info struct, arch-neutral kernel entry, serial console ops, banner |
| 0.5 | UEFI loader | Our loader boots the x86_64 kernel stub under QEMU with UEFI firmware |
| 0.6 | RISC-V firmware | Our machine-mode firmware boots the RISC-V kernel stub under QEMU |
| 0.7 | Boot tests in CI | Scripted QEMU boot tests for both ports, run on every push |

Later phases get their slice lists when the phase before them closes (Q4, Q7).

## 1.0 checklist (end of phase 6)

- Networking: our own stack, DHCP, name lookup, the four first tools, time sync (N1 to N11)
- USB keyboard, mouse and storage through xHCI (H7)
- Audio through virtio sound (H11)
- NVMe, FAT, swap, yartfs v2 with checksums and crash safety (H6, F4, M9, F2)
- ACPI interpreter in, with shutdown, battery and backlight on a real laptop (H5, H15)
- Boots from USB on one real x86 machine (D9)
- Boots in VirtualBox, VMware and Hyper-V (R8)
- Package manager with dependency resolution, and a ports tree (U15, R4)
- First ports: Lua, a small C compiler, DOOM, Python (R2)
- Installer and developer kit (G4, R7)
- Kernel address randomization, TLS-grade cryptography, SSH (B11, X8, N10)

## After 1.0

Stretch goals, none planned in detail: the custom CPU project, GPU acceleration and native GPU drivers (H9, H10), compositor blur (W6), Linux binary compatibility (G7), a web browser (G3), C++ (R6), self-hosting (R3), Wi-Fi (H13).
