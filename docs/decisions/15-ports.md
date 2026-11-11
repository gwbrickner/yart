# Software and ports

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| R1 | When do ports of existing software start? | Phase 3 · Phase 6 | Phase 6, following D7. | Claude's pick, approved |
| R2 | Which ports come first? | Lua · A small C compiler · DOOM · Python | In that order. Each one stresses a different layer: libc, the loader, graphics and input, then everything at once. | Claude's pick, approved |
| R3 | Is self-hosting a goal? | Required for 1.0 · Stretch goal · No | Stretch goal after 1.0. It needs a full compiler and build tools ported. | Claude's pick, approved |
| R4 | How are ports built? | By hand · Recipe files cross-compiled on the host | Recipe files in the repo: source, patches, build steps, license. | Claude's pick, approved |
| R5 | When a port fails to build, what gets fixed? | Patch the port · Fix our libc | Fix our libc unless the port is doing something wrong. Patches stay small. | Claude's pick, approved |
| R6 | Is C++ supported? | Before 1.0 · After 1.0 | After 1.0. | Claude's pick, approved |
| R7 | Is there a developer kit for outside apps? | Yes, in phase 6 · No | Yes: headers, libraries and toolkit docs in one download. | Claude's pick, approved |
| R8 | Which hypervisors are supported? | QEMU only · VirtualBox, VMware and Hyper-V too | QEMU, VirtualBox, VMware and Hyper-V. VirtualBox by phase 5 (D9), the others in phase 6. | Graeson, detailed by Claude |
| R9 | How are licenses of ported software handled? | Not tracked · Recorded per recipe | Recorded per recipe. Copyleft software ships as separate packages, never mixed into our code. | Claude's pick, approved |
