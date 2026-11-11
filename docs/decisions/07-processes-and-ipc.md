# Processes, scheduling and IPC

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| S1 | How are processes created? | fork and exec · Spawn only · Both | Spawn is the native call. fork and exec exist for the POSIX layer. | Claude, following D6 |
| S2 | How are threads implemented? | One kernel thread per user thread · User-level threads | One kernel thread per user thread. | Claude's pick, approved |
| S3 | Which scheduling algorithm? | Round robin · Multilevel feedback queue · Fair-share | Multilevel feedback queue. It keeps interactive programs responsive with little code. | Claude's pick, approved |
| S4 | What priority controls exist? | Nice levels · A fixed-priority class · Both | Both. The fixed class is for audio and the compositor. | Claude's pick, approved |
| S5 | How long is a time slice? | Fixed · Tunable | 10 ms to start, tunable, adjusted from benchmarks. | Claude's pick, approved |
| S6 | How is work spread across cores? | One global queue · Per-CPU queues with work stealing | Per-CPU queues with work stealing. | Claude's pick, approved |
| S7 | Are there signals? | Full POSIX · The subset shells and ports need · A different mechanism | The subset shells and ports need: interrupt, terminate, kill, child exit, stop and continue, fault. | Claude's pick, approved |
| S8 | Are there process groups, sessions and controlling terminals? | Yes · No | Yes. Shell job control depends on them. | Claude's pick, approved |
| S9 | What executable formats load? | ELF only · ELF plus `#!` scripts | ELF plus `#!` scripts. | Claude's pick, approved |
| S10 | Static or dynamic linking? | Static only · Dynamic from phase 3 · Dynamic from phase 5 | Static through phase 4. The dynamic linker arrives in phase 5, when the GUI toolkit makes shared libraries worth it. | Claude's pick, approved |
| S11 | What do userland locks block on? | A futex-style wait call · Kernel mutex objects | A futex-style wait call. | Claude's pick, approved |
| S12 | Which IPC mechanisms exist? | Pipes · Local sockets with descriptor passing · Shared memory · Message ports | Pipes in phase 2. Local sockets and shared memory before the GUI. No message bus until something needs one. | Claude's pick, approved |
| S13 | How does a program wait on many things at once? | `poll` · A scalable event queue | `poll` first, one event queue API in phase 5. | Claude's pick, approved |
| S14 | How does userland inspect processes? | A `/proc` filesystem · Dedicated syscalls | A small `/proc`. | Claude's pick, approved |
| S15 | What happens when a program crashes? | Silent exit · Report with backtrace in the system log · Core dump file | Report with backtrace in the system log. Core dumps in phase 6. | Claude's pick, approved |
| S16 | Are there resource limits? | Open files and memory per process · None | Open files and memory per process, added in phase 3. | Claude's pick, approved |
