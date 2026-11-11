# Decide first

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| D1 | Is this a new OS or bongOS's next era? | Fresh repo · Evolve bongOS in place · Fresh repo that lifts proven bongOS code | Fresh repo. Nothing is carried over from bongOS or NexOS; all code is written new. | Graeson |
| D2 | What does "the OS is done" mean, so the CPU project can start? | Command line complete (phase 3) · Usable desktop in QEMU (phase 5) · Boots on real hardware (phase 6) | The OS is done (1.0) when phase 6 is complete and it boots in VirtualBox. The custom CPU project starts after that. | Graeson |
| D3 | Which architectures, in what order? | x86\_64 only at first · x86\_64 + RISC-V 64 together · Add AArch64 from the start too | x86\_64 and RISC-V 64 from the first commit, AArch64 third in phase 4. Two keeps the port layer honest without tripling early work. | Claude's pick, approved |
| D4 | What language is the kernel written in? | C · Rust · C kernel with Rust allowed in userland | C. You know it, every tutorial and reference kernel uses it, and one clang build targets all three architectures. | Claude's pick, approved |
| D5 | What kind of kernel? | Monolithic · Microkernel · Hybrid (drivers in kernel, GUI and services in userspace) | Monolithic, with the compositor and services in userspace. Microkernels cost a lot of IPC design before anything works. | Claude's pick, approved |
| D6 | How Unix-like is it? | POSIX-shaped throughout · Own API with a POSIX layer on top · Fully original | Our own native system API, with a POSIX layer on top. Ordinary POSIX C source should compile and run unchanged against the POSIX layer. | Graeson, detailed by Claude |
| D7 | What third-party code is allowed in the tree? | None at all · Bootloader and firmware only · Also libc, fonts, compression libraries | No third-party code in the tree, including bootloaders and firmware. Approved exceptions: the math library (A10), the ACPI interpreter (H5), cryptography (X8), text shaping (W15), and phase 6 ports, which live as separate packages. | Graeson, detailed by Claude |
| D8 | What kind of desktop? | Classic stacking windows · Tiling · Hybrid with both | Classic stacking window manager on our own toolkit, software rendered. The snap and keyboard shortcuts in W19 still apply. | Graeson |
| D9 | What is the first real machine it must boot on? | QEMU only · Your desktop PC · Your laptop · A RISC-V dev board | QEMU through phase 4. Booting in VirtualBox is required by the end of phase 5. One real x86 machine, booted from USB, in phase 6. | Graeson, detailed by Claude |
| D10 | Who writes the code, and how much of it do you want to understand line by line? | Claude Code sessions build from specs · You write, Claude reviews · Split by subsystem | Spec first: each milestone gets a written spec and tests before it is built. Claude Code agents write essentially all the code. Graeson approves specs and reads merges. | Graeson, detailed by Claude |
