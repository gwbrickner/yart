# Syscalls, ABI and libc

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| A1 | Where does libc come from? | Our own · Port mlibc · Port musl | Our own, in two layers: a native system library exposing the native API (D6), and a POSIX libc built on top of it. | Claude, following D6 |
| A2 | How much of POSIX do we implement? | Full conformance · Grown by need | Grown by need. Each port's missing functions set the next batch. | Claude's pick, approved |
| A3 | What is the stable interface? | Raw syscall numbers · The libc API only | The native library and the POSIX layer are the stable interfaces. Raw syscall numbers may change until 1.0. | Claude, following D6 |
| A4 | How are syscalls defined? | Hand-written in three places · One table that generates kernel dispatch, libc stubs and docs | One table. | Claude's pick, approved |
| A5 | Which userland calling convention? | Each architecture's standard ABI · Custom | Each architecture's standard ABI, so clang's defaults just work. | Claude's pick, approved |
| A6 | How does clang target the OS? | Generic ELF target plus our sysroot · A custom OS target | Generic ELF target plus our sysroot now. A proper OS target when a compiler gets ported in phase 6. | Claude's pick, approved |
| A7 | How wide are time and file offsets? | 64-bit · Platform dependent | 64-bit on every port. | Claude's pick, approved |
| A8 | What text encoding? | UTF-8 everywhere · Locale dependent | UTF-8 everywhere. | Claude's pick, approved |
| A9 | What are the path limits? | 4096 path, 255 name · Other | 4096 bytes per path, 255 per name. | Claude's pick, approved |
| A10 | Where does the math library come from? | Our own · A ported permissive-license one | A ported one. Correct floating-point math is specialist work, so it is an approved exception to D7. | Claude's pick, approved |
| A11 | Which thread API? | POSIX threads subset · Our own | POSIX threads subset. | Claude's pick, approved |
| A12 | Is there a syscall tracing tool? | Yes, in phase 3 · Later | Yes, in phase 3. It pays for itself the first time a port misbehaves. | Claude's pick, approved |
| A13 | Do headers use the standard POSIX names and layout? | Yes · Own layout | Yes, so ported code compiles unmodified. | Claude's pick, approved |
