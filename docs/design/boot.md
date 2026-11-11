# Boot

Status: draft      Implements: B1 to B10, P5, P6, P12, D7, H14

## Purpose
Boot covers everything between machine reset and the first line of `kernelMain`: our UEFI loader on x86_64, our machine-mode firmware on RISC-V, and the one struct both hand to the kernel. The code lives in `boot/` and shares its parsers with the kernel through `lib/`.

## Decisions applied
- B1, D7: the loader is our own PE/COFF UEFI application with our own UEFI headers. No gnu-efi or EDK2 code. The UEFI firmware QEMU runs (P12) comes from the container image (T6) and is not tree code.
- B2, P6: our RISC-V firmware is the whole chain and stays resident to answer SBI calls.
- B7, B11, Q8: one ELF loader in `lib/`, shared by both boot paths and fuzzed on the host. No relocation until phase 6.

## Design

### Handoff contract
Every boot path ends in the same state: boot CPU only, interrupts off, paging on, kernel segments mapped at their link addresses with their ELF permissions, all RAM in the direct map (M12, large pages per M11), and a direct-map pointer to `BootInfo` in the first argument register. Control arrives at the ELF entry `archEntry`, an assembly stub in `arch/<port>` (T12).

### BootInfo (B5)

```c
#define BOOT_INFO_MAGIC 0x54524159u    /* "YART" */

typedef enum BootMemKind {
    BOOT_MEM_USABLE, BOOT_MEM_RESERVED, BOOT_MEM_KERNEL, BOOT_MEM_INIT_FS,
    BOOT_MEM_BOOT_DATA, BOOT_MEM_FIRMWARE_TABLES, BOOT_MEM_FIRMWARE_RUNTIME,
} BootMemKind;

typedef struct BootMemRegion {
    uint64_t base, length;
    uint32_t kind, reserved;
} BootMemRegion;

typedef struct BootFramebuffer {
    uint64_t base;
    uint32_t width, height, pitch, bitsPerPixel;
    uint8_t  redShift, greenShift, blueShift, reserved[5];
} BootFramebuffer;

typedef enum BootTableKind {
    BOOT_TABLE_NONE, BOOT_TABLE_ACPI_RSDP, BOOT_TABLE_DEVICE_TREE,
} BootTableKind;

typedef struct BootInfo {
    uint32_t magic, version;
    uint64_t memRegions, memRegionCount;
    BootFramebuffer framebuffer;
    uint64_t commandLine;
    uint64_t initFsBase, initFsSize;
    uint64_t firmwareTable;
    uint32_t firmwareTableKind, reserved;
    uint64_t firmwareRuntime;
    uint64_t bootCpuId;
} BootInfo;
```

Addresses are physical and fields are fixed-width (P1): the UEFI loader is compiled for a different ABI than the kernel.

### Why one struct
Each machine describes itself differently. UEFI hands over a memory map in its own format, a graphics protocol and a list of tables. A RISC-V machine hands over a device tree and nothing else. If the kernel read those directly, `kernel/` would need one startup path per firmware, which principle 1 forbids.

So each boot path fills in the same form before it jumps, and the kernel reads only the form:

- `magic`, `version`: prove the pointer is a `BootInfo` and that loader and kernel match.
- `memRegions`: which physical memory is free and which is taken. The page allocator starts from this.
- `framebuffer`: where the screen's pixels live and how they are laid out. `base` is zero when there is no screen.
- `commandLine`: the option string (B9).
- `initFsBase`, `initFsSize`: where the tar archive of first files sits in RAM.
- `firmwareTable`: the ACPI root pointer or the device tree, for hardware discovery (P7).
- `firmwareRuntime`: the UEFI system table, so the x86 port can ask UEFI to power off or reboot (H14). Zero elsewhere.
- `bootCpuId`: which CPU booted. A RISC-V kernel cannot read its own hart ID.

The kernel entry is the same three steps on every port: `archEntry` sets a stack, registers the port's ops tables (see arch-contract.md), and calls `void kernelMain(const BootInfo *bootInfo)`. Only the register carrying the pointer differs: `rdi`, `a0`, `x0`.

### Command line, console, initial filesystem
- B9: one UTF-8 string of space-separated `key=value` words: `root=`, `console=`, `loglevel=`, `init=`. Boot code passes it through untouched; `lib/` parses it.
- B10: the early console is serial, 115200 8N1, behind the port's early console ops.
- B8: the initial filesystem is a ustar archive built by a host tool and loaded whole at a page-aligned address. From phase 2 the kernel unpacks it into tmpfs.

### Higher-half layout (B6)
The kernel image is linked in the top 2 GiB (`0xFFFFFFFF80000000`) on every port. The direct map starts at the bottom of the higher half (`0xFFFF800000000000` with 48-bit addresses). The ranges between belong to the memory design note.

## Per-architecture notes

### x86_64: UEFI loader
Built by clang for `x86_64-unknown-windows`, freestanding, and linked by lld as a PE/COFF EFI application, `BOOTX64.EFI`. `boot/uefi/efi.h` declares only what we call and is written from the UEFI specification. Steps:

1. Open the loader's own volume and read `\yart\kernel.elf`, `\yart\initfs.tar` and `\yart\cmdline.txt` into allocated pages.
2. Load the ELF segments and zero the bss.
3. Record the current GOP mode as the framebuffer.
4. Find the ACPI RSDP in the configuration table.
5. Build page tables: kernel, direct map, and an identity map of the loader for the switch.
6. Get the memory map and call `ExitBootServices`, retrying with a fresh map if the key is stale. Convert the map to `BootMemRegion` entries.
7. Load the new tables and jump to `archEntry` using the System V calling convention.

### riscv64: machine-mode firmware
QEMU `virt` with `-bios` starts every hart in our firmware at the base of RAM, in machine mode, with the hart ID in `a0` and the device tree in `a1`. The firmware:

1. Sets a stack and trap vector per hart. One hart continues; the rest park.
2. Reads RAM, UART, CLINT and reset device addresses from the device tree.
3. Delegates supervisor-level traps and interrupts, opens the `time` counter, and uses PMP to protect itself.
4. Finds the boot archive, a tar holding `kernel.elf` and `initfs.tar`. QEMU's `-kernel boot.tar` copies a non-ELF file into RAM unchanged and reports its address in the structure passed in `a2`. The command line comes from `/chosen/bootargs` (`-append`).
5. Loads the ELF, builds page tables, fills `BootInfo`.
6. Sets `satp`, sets the previous privilege to supervisor, and executes `mret` to `archEntry`.

It then serves these SBI extensions: Base, Debug Console (early serial), System Reset (H14) from phase 0; Timer (K5) from phase 1; IPI and Hart State Management in phase 4.

### aarch64 (plan, confirmed in phase 4)
B3: rebuild the UEFI loader for `aarch64-unknown-windows` as `BOOTAA64.EFI`, take the device tree from the UEFI configuration table (P7), use the PL011 for the early console and PSCI for reset (H14).

## Build order
- 0.4: `BootInfo`, `archEntry`, `kernelMain`, early console, banner.
- 0.5: UEFI loader. 0.6: RISC-V firmware with Base, Debug Console, System Reset.
- Phase 1: SBI Timer. Phase 4: IPI, hart management, AArch64 loader.

## Test plan
- Host unit tests and fuzzing (Q1, Q8): ELF loader, tar reader, command-line parser, device tree reader, memory-map conversion.
- In-kernel self test: `BootInfo` is valid (magic, sorted regions, readable archive).
- Boot tests (0.7): each port reaches the banner in QEMU. Boot time is recorded (B13).

## Risks and open questions
- B5, H14, P6: `firmwareRuntime`, `bootCpuId`, `magic` and `version` are not in B5's field list and need approval. Keeping UEFI runtime code mapped for H14 is not designed.
- B2, F13: the RISC-V firmware has no storage driver, so QEMU preloads the boot archive. Booting from the F13 disk image is unscheduled. The `a2` handoff must be checked against the pinned QEMU.
- P6, B5: under standard firmware something must still build `BootInfo`. Our loader half would have to run in supervisor mode, which no slice plans.
- F4, B1, F13, D9: the EFI partition is FAT, but FAT arrives in phase 6. Phase 0 can use QEMU's directory-backed FAT drive; VirtualBox in phase 5 needs a real image.
- M13, P3, B6: Sv39 or Sv48 on RISC-V is undecided. It sets the direct-map base and the table depth.
- P7, roadmap: device tree parsing is listed for phase 4, but the firmware needs a reader in slice 0.6.
- B9: where the x86 command line is stored and the `console=` value syntax are not settled.
- T1: the loader needs `__attribute__((sysv_abi))`. The allowed-extension list is not written yet.
