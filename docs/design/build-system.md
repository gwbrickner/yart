# Build system

Status: draft      Implements: T1 to T14

## Purpose
The build system turns the monorepo into a boot program, kernel, userland and bootable image for each port, plus host tools and host tests. It also enforces the mechanical rules: language level, warnings, bans, formatting. Everything goes through `./x`.

## Decisions applied
- T2, T3, T4: one LLVM toolchain. clang compiles C and `.S` files, LLD links, `llvm-ar` and `llvm-objcopy` do the rest.
- T5: the cross file is `arch/<port>/cross.ini`, so a port stays one directory plus a boot path (P8).
- D7: the toolchain is a host tool; none of its runtime libraries is linked into a yart binary (reading to confirm).

## Design

### The `./x` wrapper
```
./x build <port|host|all>     configure if needed, then build
./x run <port> [--gdb]        build, make the image, boot QEMU on serial (G4)
./x test [host|<port>|all]    run tests; no argument means all
./x image <port>              write the bootable image (F13)
./x fmt                       clang-format the tree
./x lint                      format check, clang-tidy, tree checks
./x config <port>             open the configuration menu
./x fuzz, ./x bench           see testing.md
./x doctor                    compare installed tools with the pins
```

### Build directories and targets
One Meson project (T13), configured once per directory: `build/host` (native, sanitizers on) and `build/<port>` (cross). A port build produces three link formats (B1, B2, B7):

- the PE/COFF loader (`BOOTX64.EFI`) or raw firmware (`firmware.bin`) from `boot/`;
- `kernel.elf` from `kernel/`, `arch/<port>/`, `drivers/`, `lib/` and `fs/`, drivers linked in (K1);
- static userland programs (S10), packed into the initial tar (B8);
- host tools from `tools/`, built natively, which write the image.

`lib/` and `fs/yartfs/` compile in three flavours: kernel, userland (generic ELF target plus our sysroot, A6) and host. The cross file names only tools and target; flag sets (`kernelArgs`, `userArgs`, `loaderArgs`) are Meson variables in `arch/<port>/meson.build`.

### Flags
- Everywhere: `-std=c17 -Wall -Wextra -Werror` (T8). Proposed additions: `-Wpedantic -Wvla -Wshadow -Wundef -Wstrict-prototypes`.
- Kernel side: `-ffreestanding -nostdlib -fno-omit-frame-pointer` (K9), a `-Wframe-larger-than=` limit, stack protector from phase 2 (X6).
- Sanitizers (T9): host builds use address and undefined-behavior checks. Debug kernels use `-fsanitize=undefined` from phase 1, with our own handlers.

### Allowed extensions (T1, proposed)
- Attributes: `packed`, `aligned`, `section`, `used`, `unused`, `noreturn`, `noinline`, `always_inline`, `format`, `warn_unused_result`, `fallthrough`, `no_sanitize`.
- Builtins: `__builtin_expect`, `unreachable`, bit counts, byte swaps, checked arithmetic, frame and return address.
- `__typeof__` and statement expressions, only in `lib/` macros.
- `__asm__`, only where the assembly list allows.

Everything else is out, including `__int128`, zero-length arrays, computed goto and `#pragma once`. `-Wpedantic` catches most of it and lint the remainder; the switches exempting the allowed items are to verify in slice 0.2. The list needs Graeson's approval.

### Banned constructs (T11)
Each ban has a mechanical check, applied to `kernel/`, `arch/`, `drivers/`, `boot/` and the kernel flavour of `lib/` and `fs/`.

- Floating point: on x86_64, `-mgeneral-regs-only` (to verify). On riscv64, `-march=rv64ima` with no soft-float library, so any use fails to link. Lint also rejects `float` and `double`.
- Variable-length arrays: `-Werror=vla`.
- Unbounded recursion: clang-tidy `misc-no-recursion`; an exception needs a suppression comment stating the bound.
- Allocation in interrupt handlers: no static check. Allocator entry points assert they are not in interrupt context (K11), and a self test proves the assertion fires.

### Formatting and static analysis (T7, T10)
`.clang-format` fixes layout. `.clang-tidy` carries a small set: `readability-identifier-naming` for T7 (off in `user/libc/`), a few `bugprone-*` and `clang-analyzer-core*` checks, and `misc-no-recursion` on the kernel side. `./x lint` adds two tree checks: architecture macros outside `arch/` and `boot/` (principle 1), and the assembly list.

### Assembly allow-list (T12)
```
# tools/lint/asm-allow.txt
boot/riscv-fw/entry.S   reset entry
boot/riscv-fw/trap.S    machine-mode trap entry and exit
arch/*/entry.S          kernel entry
arch/*/trap.S           trap and syscall entry and exit
arch/*/switch.S         context switch
arch/*/cpu.h            inline CPU primitives
```
Assembly outside this list fails lint.

### Configuration (T14)
```
kconfig                     root menu, sources the files below
kernel/kconfig  drivers/kconfig  arch/<port>/kconfig
configs/<port>.config       committed defaults, non-default values only
build/<port>/.config        resolved configuration, not committed
build/<port>/generated/config.h
```
`./x config <port>` edits `.config`; a missing one is created from `configs/<port>.config`. Each build regenerates `config.h` (`#define CONFIG_DEBUG_UBSAN 1`). Meson reads `.config` at configure time to choose source files (its `keyval` module, to verify) and passes `-include generated/config.h` to kernel-side targets. Debug features (T9, K3, M16) are options here.

The front end is a host tool with a fixed contract: menu files and `.config` in, `.config` and `config.h` out. Sourcing options, not decided here:
1. An existing Python implementation installed into the container.
2. The Linux kernel's own tools, built on the host.
3. Our own front end in `tools/kconfig/`, or only our own generator with an external menu editor.

### Pinned environment (T6)
- `tools/versions.txt` is the one list of tool versions.
- `tools/container/Containerfile` builds the image from Ubuntu 24.04 pinned by digest, the release cloud sessions run. CI builds only inside it.
- `./x` runs inside the image when a container runtime is present, otherwise natively once `./x doctor` passes. Windows uses WSL 2.
- `scripts/setup-dev.sh` makes a fresh cloud session usable. It is idempotent, quick when nothing is missing, installs through apt or PyPI, and exits non-zero naming anything it could not install. It runs from the cloud environment's setup script or a SessionStart hook; slice 0.1 picks one.

Needed beyond clang and Ninja: LLD, LLVM binutils, clang-format, clang-tidy, Meson, QEMU system emulators, UEFI firmware for QEMU, GDB. Package names and versions: to verify in slice 0.1. One cloud session on 2026-11-10 had clang 18.1.3 and LLD but no Meson or QEMU.

## Per-architecture notes
- x86_64: `-march=x86-64` (P4); kernel adds `-mno-red-zone -mcmodel=kernel`. The loader compiles for the COFF target and links through LLD's COFF driver as an EFI application. The image is GPT (F5) with a FAT partition.
- riscv64: kernel is `-march=rv64ima -mabi=lp64` (P3). The firmware links as ELF, then `llvm-objcopy -O binary` makes `firmware.bin` for QEMU's `-bios`.
- aarch64 (phase 4): a third cross file; boot per B3.

## Build order
- 0.1: layout, container, `setup-dev.sh`, format and tidy configuration, a lint script for CI.
- 0.2: cross files, `./x` with `build`, `config`, `fmt`, `lint`, `doctor`; generated header; ban checks.
- 0.3: `build/host` with sanitizers.
- 0.4 to 0.7: link scripts, PE/COFF and raw outputs, `run`, `image`, `test`.
- Later: UBSan handlers (phase 1), syscall generator and sysroot (phase 2), aarch64 and loadable modules (phase 4), shared libraries (phase 5).

## Test plan
- Host: golden-file tests for `.config` to `config.h`.
- Rule fixtures in `tests/host/build-rules/`: one source per ban and per disallowed extension, passing only when the compiler or lint rejects it.
- CI runs `scripts/setup-dev.sh` on a clean Ubuntu 24.04 image, then `./x test host`.
- Boot tests prove the link formats end to end.

## Risks and open questions
- T14, D7: where the Kconfig front end comes from (open question 2). Blocks slice 0.2.
- T5, B1: "one cross file per architecture" meets a loader that needs a COFF target while the kernel needs ELF. Either a per-target override in one build directory, or a second cross file, which needs T5 confirmed. To verify in 0.2.
- T7: clang-format cannot check identifier case. This note uses clang-tidy; T7's text needs an edit.
- T9: the address sanitizer has no runtime for yart userland. Either we write one, or "userland" means userland code built on the host.
- D7: clang emits calls to runtime helpers such as `memcpy`. This note supplies them from `lib/`; D7 does not say whether linking the compiler's runtime library counts.
- P3: the RISC-V userland baseline and float ABI are unset; if they differ from the kernel's, `lib/` builds per ABI.
- T12, T3: the list omits userland assembly (program entry, syscall stubs), and no x86 assembly syntax is chosen.
- T11: one ban has only a runtime check, the tidy check bans all recursion, and coverage of `boot/` is assumed.
- F13, F4, P5: UEFI needs a FAT partition from slice 0.5, but FAT is phase 6 and F13 requires our own tools. Either a host-side FAT writer in phase 0, or QEMU's directory-backed FAT drive until one exists.
- T6: container runtime, registry and CI provider are unnamed. QEMU's UEFI firmware is a third-party binary used only on the host.
- D4: the language of `./x` and host scripts is unset; Python 3 is proposed.
- Roadmap: CI lint starts in 0.1 but `./x` arrives in 0.2, so 0.1 uses a plain script.
