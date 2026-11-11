# Boot

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| B1 | Which bootloader on x86\_64? | Limine · GRUB · Own UEFI loader | Our own UEFI loader. | Graeson |
| B2 | What is the RISC-V boot chain? | OpenSBI straight into the kernel · OpenSBI, then U-Boot · Limine | Fully custom. Our machine-mode firmware (P6) loads and enters the kernel. No OpenSBI or U-Boot. | Graeson, detailed by Claude |
| B3 | What is the AArch64 boot chain? | Limine on UEFI · Direct boot with a device tree | Fully custom. The plan is our UEFI loader rebuilt for AArch64; the exact chain is confirmed in phase 4. | Graeson, detailed by Claude |
| B4 | Do we ever write our own bootloader? | Never · Side project after phase 5 | Yes, from phase 0. The UEFI loader and the RISC-V machine-mode firmware are part of the tree. | Graeson |
| B5 | How does the kernel receive boot data? | One common boot-info struct filled by each port · Kernel reads each protocol directly | One common struct. Each boot path gathers what its platform knows and fills the same struct: memory map, framebuffer, command line, initial filesystem location, firmware table pointer. The kernel's entry point takes only that struct, so nothing above `arch/` knows which loader ran. See docs/design/boot.md. | Claude pick, explanation requested |
| B6 | Where does the kernel live in memory? | Higher half · Identity mapped | Higher half on every port. | Claude's pick, approved |
| B7 | What format is the kernel image? | ELF · Flat binary | ELF everywhere. | Claude's pick, approved |
| B8 | What format is the initial filesystem? | tar · cpio · Custom | tar. Host tools can build and inspect it. | Claude's pick, approved |
| B9 | Is there a kernel command line? | Yes, key=value · No | Yes: `root=`, `console=`, `loglevel=`, `init=` to start. | Claude's pick, approved |
| B10 | What is the first console? | Serial · Framebuffer text | Serial on every port. The framebuffer console arrives in phase 3. | Claude's pick, approved |
| B11 | Is the kernel's load address randomized? | From the start · Phase 6 · Never | Phase 6. | Claude's pick, approved |
| B12 | What does boot look like? | Full log · Quiet with a splash | Full log until the GUI exists, then a quiet option. | Claude's pick, approved |
| B13 | Is boot time tracked? | Tracked in CI with a target · Ignored | Tracked in CI. Regressions show up as a number, not a feeling. | Claude's pick, approved |
