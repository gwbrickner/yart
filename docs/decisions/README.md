# Decisions

Every design decision for yart, settled on 2026-11-10. 248 decisions in 17 areas. These files are the source of truth; a spec or design note that contradicts one is wrong.

| File | Area | IDs | Count |
| --- | --- | --- | --- |
| [00-decide-first.md](00-decide-first.md) | Decide first | D1 to D10 | 10 |
| [01-identity-and-goals.md](01-identity-and-goals.md) | Identity and goals | G1 to G11 | 11 |
| [02-platforms.md](02-platforms.md) | Platforms and portability | P1 to P14 | 14 |
| [03-toolchain-and-build.md](03-toolchain-and-build.md) | Language, toolchain and build | T1 to T14 | 14 |
| [04-boot.md](04-boot.md) | Boot | B1 to B13 | 13 |
| [05-kernel-core.md](05-kernel-core.md) | Kernel core | K1 to K14 | 14 |
| [06-memory.md](06-memory.md) | Memory | M1 to M16 | 16 |
| [07-processes-and-ipc.md](07-processes-and-ipc.md) | Processes, scheduling and IPC | S1 to S16 | 16 |
| [08-syscalls-abi-libc.md](08-syscalls-abi-libc.md) | Syscalls, ABI and libc | A1 to A13 | 13 |
| [09-storage.md](09-storage.md) | Storage and filesystems | F1 to F16 | 16 |
| [10-drivers.md](10-drivers.md) | Drivers and hardware | H1 to H17 | 17 |
| [11-security.md](11-security.md) | Security and users | X1 to X10 | 10 |
| [12-userland.md](12-userland.md) | Command line and userland | U1 to U18 | 18 |
| [13-gui.md](13-gui.md) | GUI | W1 to W30 | 30 |
| [14-networking.md](14-networking.md) | Networking | N1 to N12 | 12 |
| [15-ports.md](15-ports.md) | Software and ports | R1 to R9 | 9 |
| [16-quality-and-workflow.md](16-quality-and-workflow.md) | Testing, quality and workflow | Q1 to Q15 | 15 |

## How to use these

- Cite decisions by ID (`K5`, `W13`) wherever a choice depends on one.
- If a decision blocks good work or two decisions conflict, stop and raise it. Do not work around it silently.
- The `Decided by` column says whose call it was. `Graeson` rows were his explicit answer. `Graeson, detailed by Claude` rows are his answer with the consequences written out. `Claude, following X` rows changed because another decision changed.

## Changing a decision

1. Graeson approves the change.
2. Edit the row in place and add a line to [CHANGELOG.md](CHANGELOG.md): date, ID, old decision, new decision, reason.
3. Update every spec and design note that cites the ID.

Items still waiting on an answer are in [../open-questions.md](../open-questions.md).
