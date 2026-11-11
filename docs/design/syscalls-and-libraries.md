# Syscalls and libraries

Status: draft      Implements: D6, G7, A1 to A13, K7, K12, K14, T7

## Purpose
This note defines the boundary between programs and the kernel: the native system API, the one table every syscall comes from, the two libraries above it, the per-port entry path, and the personality hook. The native API is the real interface (D6).

## Decisions applied
- D6, principle 9: libc never traps into the kernel; it calls libyart.
- A3: programs depend on library functions, never on numbers.
- K7: a call returns a non-negative result or a negative error code. There is no native `errno`.

## Design

### Native API style
- Names are `objectVerb` in camelCase (T7): `fileOpen`, `processSpawn`.
- Every kernel object a program holds is a descriptor (K12). Calls that work on any descriptor start with `fd`.
- Every call returns `int64_t`. Anything else uses an out pointer.
- Paths are UTF-8, NUL-terminated, at most 4096 bytes (A8, A9), and resolved relative to a directory descriptor.
- Times and offsets are 64-bit; time is nanoseconds (A7, F14).
- Argument structs use fixed-width fields (P1) and start with `size` so they can grow. Unknown flag bits are an error.

```c
int64_t fileOpen(int dirFd, const char *path, uint32_t flags, uint32_t mode);
int64_t fileSeek(int fd, int64_t offset, uint32_t whence, int64_t *positionOut);
int64_t memMap(const MapArgs *args, void **addressOut);
```

| Area | Calls |
| --- | --- |
| Descriptors | `fdRead` `fdWrite` `fdClose` `fdDuplicate` `fdStat` `fdPoll` `fdControl` |
| Files | `fileOpen` `fileSeek` `fileRemove` `dirRead` |
| Memory | `memMap` `memUnmap` `memProtect` `memObjectCreate` |
| Processes, threads | `processSpawn` `processWait` `processSignal` `processExit` `threadCreate` `threadSleep` `futexWait` `futexWake` |
| Signals | `signalAction` `signalMask` `signalReturn` |
| IPC, events | `pipeCreate` `socketCreate` `socketSend` `socketReceive` `eventQueueCreate` `eventQueueWait` |
| System | `clockRead` `sandboxRestrict` |
| POSIX support | `processFork` `processExec` `processWaitAny` `processKill` `processSetGroup` `sessionCreate` |

### The syscall table (A4)
`kernel/syscall/syscalls.def` uses the system configuration syntax (U13), so the parser in `lib/` is shared. One section per call:

```
[fileOpen]
number  = 12
group   = file
layer   = native
since   = 2
args    = int dirFd, path path, uint32_t flags, uint32_t mode
returns = fd
errors  = ERR_NOT_FOUND, ERR_ACCESS, ERR_NAME_TOO_LONG
doc     = Open or create the file at path, relative to dirFd.
```

`layer` is `native` or `posix`; `since` is the phase. An argument is a scalar C type or a user-memory kind: `path`, `in(Type)`, `out(Type)`, `inBytes(length)`, `outBytes(length)`. At most six arguments, each one register. `errors.def` lists each code once with its native and POSIX names (`ERR_NOT_FOUND`, `ENOENT`) and message.

The host tool `tools/syscall-gen` generates:
- Kernel: the dispatch table, typed handler prototypes (`sysFileOpen`), thunks that unpack six registers.
- libyart: number constants, public headers with doc comments (K13), C stubs over one raw per-port entry.
- Docs: one manual page per call (U14).
- Tools: tracer decode table (A12), fuzzer argument kinds (Q8), sandbox groups (X7).
- A hash of the table, placed in an ELF note in every program.

### Two libraries (A1)
libyart holds program start-up, the generated stubs and the native headers under `<yart/>`. libc holds ISO C and POSIX under the standard header names (A13) and sets `errno` from the negative result. Both install into the sysroot (A6) and are static until phase 5 (S10).

| POSIX | Native calls | Library work |
| --- | --- | --- |
| `open` | `fileOpen` | flag translation |
| `read`, `write` | `fdRead`, `fdWrite` | none |
| `stat` | `fileOpen`, `fdStat` | struct conversion |
| `readdir` | `dirRead` | buffering |
| `malloc` | `memMap`, `memUnmap` | size-class allocator (M15) |
| `posix_spawn` | `processSpawn` | file actions become descriptor actions |
| `fork` | `processFork` | none |
| `waitpid` | `processWaitAny` | status encoding |
| `poll`, `select` | `fdPoll` | set conversion |
| `pthread_create` | `memMap`, `threadCreate` | stack and thread-local block |
| `pthread_mutex_lock` | `futexWait` when contended | atomics in user space |
| `clock_gettime` | `clockRead` | unit conversion |

Not pure translation; the kernel must implement these:
- Fork (M6) and exec in place, keeping pid and descriptors.
- Signals: asynchronous delivery, masks, interrupted calls (S7).
- Numeric pids, wait for any child, zombies, reparenting.
- Process groups, sessions, controlling terminals, the TTY line discipline (S8, U6).
- Descriptor rules: lowest free number, `dup2`, close-on-exec, offsets shared across fork and `dup`, append and non-blocking modes.
- File rules: remove while open, links (F11), owner, group, mode and `umask` (F10), a working directory that survives exec.
- Credential changes for login and the elevation service (X2, X4).

### Personalities (G7)

```c
typedef struct SyscallEntry {
    int64_t   (*handler)(const uint64_t args[6]);
    const char *name;
    uint8_t     argCount, group;
} SyscallEntry;

typedef struct Personality {
    const char         *name;
    const SyscallEntry *table;
    uint32_t            count;
    int64_t (*mapError)(int64_t error);
    int     (*buildStack)(Process *process, const ProgramImage *image);
    const SignalAbi    *signals;
} Personality;
```

The loader sets `Process.personality` from the program's ELF note; fork inherits it. Before 1.0 only `native` exists and a program without its note is refused. For a later Linux personality, these hold from phase 2:
1. Nothing bypasses `personality->table`: the tracer, sandbox and fuzzer record personality plus number.
2. Handlers are thin. `sysFileOpen` copies arguments in (K14) and calls a kernel function that knows nothing about numbers or user struct layouts.
3. The native register convention matches Linux's on each port, so trap entry and `syscallArgs` need no personality knowledge.
4. Kernel errors are K7 codes; `mapError` translates at the boundary.
5. Signal numbers, signal frames, the initial stack and user structs are personality data.
6. Sandbox promises name table groups, not numbers.

### libm, threads, tracing
- A10: libm is ported code in its own directory with its licence, built as a separate library and checked against published test vectors.
- A11: libc offers threads, mutexes, condition variables, once and keys from POSIX threads, built on `threadCreate` and the futex calls. No cancellation.
- A12: a per-process trace flag makes the dispatcher write entry and exit records (ids, personality, number, arguments, path strings, result, time) to a buffer read through a descriptor. The tool decodes them with the generated table.

## Per-architecture notes

| Port | Instruction | Number | Arguments | Result | C ABI (A5) |
| --- | --- | --- | --- | --- | --- |
| x86_64 | `syscall` | `rax` | `rdi rsi rdx r10 r8 r9` | `rax` | System V |
| riscv64 | `ecall` | `a7` | `a0` to `a5` | `a0` | LP64D (see risks) |
| aarch64 | `svc 0` | `x8` | `x0` to `x5` | `x0` | AAPCS64 |

Trap entry saves a `TrapFrame` and calls `KernelTrapOps.syscall`. The kernel reads number and arguments with `arch->context->syscallArgs`, bounds-checks the number (`ERR_NO_SYSCALL`), calls the handler and stores the result with `setReturn`. In libyart each port supplies one raw entry and the start-up stub.

## Build order
- Phase 2: table, generator, dispatch, native personality, entry on two ports, ABI note, enough of libyart to start a program.
- Phase 3: full libyart, libc, threads, tracer, syscall fuzzer.
- Phase 4: AArch64 entry.
- Phase 5: shared libraries, socket and event calls, sandbox groups.
- Phase 6: clang OS target (A6), developer kit (R7), libc grown by ports (A2).

## Test plan
- Host: generator output compared with expected files; definition parser fuzzed; libc pure functions under the address sanitizer (T9).
- In-kernel: unknown number; bad pointer for every user-memory argument kind (K14).
- Userland: one test per call on every port; struct sizes equal across ports; unmodified textbook POSIX programs compile and run (D6); thread stress; tracer output for a known program.
- Benchmark: empty syscall round trip (Q10).

## Risks and open questions
- D6, principle 9 against S1, S7, S8: POSIX-only kernel calls sit in the native table as `layer = posix`. Whether they belong to the documented native API is unsettled.
- A1, roadmap: libyart is listed in phase 3, but phase 2 programs need stubs and start-up code.
- A11, A1, M15 against D6: no native thread API is decided and `malloc` lives in libc, so native programs would need libc and `<pthread.h>`.
- A2, R1: ports start in phase 6, so only tests drive libc for three phases.
- A3, S10: static programs embed numbers, so renumbering means rebuilding everything. Whether numbers freeze at 1.0 is undecided.
- A5, A6, P3: the RISC-V float ABI is unset, and the generic ELF target's default may differ from the standard one.
- A10: which library, which phase, and its exemption from T7 and T8 are open.
- A12: the trace mechanism is a proposal. Nothing covers debugging user programs.
- K7, X7: one error code with two names, and sandbox groups in the table, are proposals.
- T12, A4: user assembly is not on the allowed list. The generator's language depends on open question 2.
- G7 with S7, S13, S14: a Linux personality also needs full signals, Linux's `/proc` and `epoll`.
