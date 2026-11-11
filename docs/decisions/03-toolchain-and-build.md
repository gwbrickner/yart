# Language, toolchain and build

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| T1 | Which C standard? | C11 · C17 · C23 | C17, with a short written list of allowed compiler extensions. | Claude's pick, approved |
| T2 | Which compiler? | clang only · GCC only · Both | clang as the one supported compiler. A GCC build in CI can come later as a portability check. | Claude's pick, approved |
| T3 | Which assembler? | NASM · clang's built-in assembler | clang's built-in assembler for every port. NASM only does x86, so it can't be the one tool. | Claude's pick, approved |
| T4 | Which linker? | LLD · GNU ld | LLD. | Claude's pick, approved |
| T5 | Which build system? | Make · CMake · Meson · Custom script | Meson with Ninja, one cross file per architecture, behind a wrapper so `./x run riscv64` is all you type. | Claude's pick, approved |
| T6 | Which development hosts are supported? | Linux only · Linux and macOS · Windows via WSL | Linux, and Windows through WSL, with a container image that pins tool versions. | Graeson |
| T7 | What naming style? | camelCase as in bongOS · snake\_case like most kernels | camelCase for our own code. POSIX-layer names match the standard. clang-format enforces it. | Graeson |
| T8 | How strict are warnings? | Warnings as errors · Warnings allowed | `-Wall -Wextra -Werror` from the first commit. | Claude's pick, approved |
| T9 | Which sanitizers? | Undefined-behavior checks in the kernel · Address checks in userland and host tests · None | Both: undefined-behavior checks in debug kernels from phase 1, address checks on host-side tests and userland. | Claude's pick, approved |
| T10 | Any static analysis? | clang-tidy · None | clang-tidy with a small rule set in CI. | Claude's pick, approved |
| T11 | What is banned in kernel code? | Floating point · Variable-length arrays · Unbounded recursion · Allocation in interrupt handlers | Ban all four. | Claude's pick, approved |
| T12 | How much assembly is allowed? | As needed · A fixed short list | A fixed list: boot entry, trap entry and exit, context switch, a few CPU primitives. Everything else is C. | Claude's pick, approved |
| T13 | One repo or several? | Monorepo · Separate repos for kernel, libc, GUI | Monorepo. | Claude's pick, approved |
| T14 | How is the kernel configured? | Kconfig-style menu system · One config header per board | A Kconfig-style menu configuration system. | Graeson |
