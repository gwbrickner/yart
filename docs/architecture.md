# Architecture overview

One page on how yart fits together. Details live in `docs/design/`; the choices behind them live in `docs/decisions/`.

## Layers

```
 apps: terminal, file manager, editor, settings          gui/apps
 desktop shell, toolkit, rasterizer, fonts               gui/
 compositor (userspace server)                           gui/compositor
 ---------------------------------------------------------------
 shell, utilities, editor, init, services                user/
 POSIX layer (libc)   <- ported C source compiles here   user/libc
 native system library (libyart)  <- the real API (D6)   user/libyart
 ============================ syscalls ==========================
 kernel core: memory, scheduler, VFS, IPC, modules       kernel/
 drivers (modules, D5/K1)                                drivers/
 arch ops tables (P8)                                    arch/<port>
 ---------------------------------------------------------------
 our UEFI loader / our RISC-V firmware (B1, B2, P6)      boot/
 machine firmware (UEFI on PCs) or bare reset vector
```

## Repository layout

```
boot/uefi/        UEFI loader (x86_64, later AArch64)
boot/riscv-fw/    machine-mode firmware and loader for RISC-V
kernel/           architecture-neutral kernel: mm/ sched/ fs/ ipc/ syscall/ time/ module/ debug/
arch/x86_64/      each port implements the ops tables and nothing else
arch/riscv64/
arch/aarch64/     (phase 4)
drivers/          bus/ virtio/ block/ input/ display/ serial/ ...
lib/              freestanding code shared by kernel, loaders and host tools
fs/yartfs/        on-disk format library, built for both the kernel and the host tools
user/libyart/     native system library
user/libc/        POSIX layer on top of libyart
user/             init/ shell/ utils/ editor/ services/
gui/              compositor/ raster/ font/ toolkit/ shell/ apps/
tools/            host tools: image builder, mkfs and fsck for yartfs, syscall table generator, Kconfig
tests/            host/ kernel/ user/ boot/
docs/             decisions/ design/ specs/ walkthroughs/
.claude/          agents, skills and settings for Claude Code
```

## The rules that hold it together

- **Arch contract (P8).** `kernel/` reaches the CPU only through ops tables each port registers at boot: CPU, MMU, interrupts, timer, context switch. A new port is a new directory under `arch/` plus a boot path.
- **Boot info (B5).** Every boot path fills one common struct and jumps to one kernel entry point. The kernel never knows which loader ran.
- **Syscalls come from one table (A4).** A single definition file generates kernel dispatch, libyart stubs and documentation.
- **Syscall personalities (G7).** Each process carries a personality that selects its syscall table. Only the native personality exists before 1.0; the hook is there so a Linux personality can be added later.
- **Two userland layers (D6, A1).** libyart is the native API in camelCase. The POSIX libc is built on libyart and carries the standard names. Our own programs use libyart; ported programs use libc.
- **Drivers are modules (K1).** Each driver has an init and exit entry and a match table (H1). They are linked into the kernel image until runtime loading lands in phase 4.
- **Shared format code.** Anything that parses bytes (yartfs, ELF, fonts, images) is a library that also compiles on the host, so it can be unit tested and fuzzed there (Q1, Q8).
- **GUI lives in userspace (W1 to W3).** Apps draw into shared-memory buffers; the compositor owns the screen, window frames and input routing.

## Naming

camelCase for our own functions, variables and struct fields; PascalCase for types; `UPPER_SNAKE` for constants and macros (T7). POSIX-layer symbols use their standard names. Files and directories are lowercase with hyphens.
