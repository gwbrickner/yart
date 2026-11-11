# Processes

Status: draft      Implements: S1 to S16, K2, K12, M6, X6

## Purpose
This subsystem owns everything that runs: processes, threads, the scheduler, signals, job control, program loading, waiting and IPC. It lives in `kernel/sched/`, `kernel/ipc/` and the loader code in `lib/`.

## Decisions applied
- S1, K12: `processSpawn` is the native call and returns a process descriptor. `processFork` and `processExec` are kernel calls kept for the POSIX layer.
- M7: file mappings arrive in phase 3, so the phase 2 loader copies segments into anonymous memory.

## Design

### Model

```c
typedef struct Process {
    int32_t            pid;
    Process           *parent;
    const Personality *personality;
    AddressSpace      *space;
    FdTable           *fds;
    Credentials        creds;
    ProcessGroup      *group;
    SignalAction       actions[SIGNAL_COUNT];
    ResourceLimits     limits;
    List               threads, children;
    int32_t            exitStatus;
} Process;

typedef struct Thread {
    int32_t      tid;
    Process     *process;
    ArchContext *context;
    TrapFrame   *frame;
    void        *userFp, *kernelStack;
    uint8_t      state, schedClass, level, baseLevel;
    uint64_t     sliceUsedNs, signalMask, pendingSignals;
    uint32_t    *exitFutex;
} Thread;
```

The scheduler sees only threads (S2). Kernel threads (K4) belong to a kernel process. `personality` selects the syscall table (G7). `exitFutex` is zeroed and woken at exit, for join.

### Creation and exit

```c
typedef struct SpawnArgs {
    uint32_t             size, flags;
    const char          *path, *workDir;
    const char *const   *argv;
    const char *const   *envp;
    const SpawnFdAction *fdActions;
    uint64_t             fdActionCount;
    int32_t              group, terminalFd;
} SpawnArgs;

int64_t processSpawn(const SpawnArgs *args);
int64_t processWait(int processFd, ProcessExit *out, uint32_t flags);
int64_t processSignal(int processFd, int signal);
int64_t threadCreate(const ThreadArgs *args);
```

A spawned child gets only the descriptors its actions (duplicate, open, close) name, unless `SPAWN_INHERIT_ALL` is set for the POSIX layer. A process descriptor becomes readable when the process exits, so `fdPoll` can wait on it. Fork copies the address space copy-on-write (M6), the descriptor table and the calling thread only. Exec replaces the image and ends the other threads. A dead process stays a zombie until its parent collects its status. Orphans go to init.

### Scheduler

```c
typedef struct RunQueue {
    SpinLock lock;
    List     fixed[SCHED_FIXED_LEVELS];
    List     normal[SCHED_NORMAL_LEVELS];
    uint32_t fixedBitmap, normalBitmap;
    Thread  *current, *idle;
    uint64_t nextBoostNs;
} RunQueue;
```

One `RunQueue` per CPU, eight levels per class to start:
1. The lowest-numbered non-empty queue runs, fixed before normal, round robin inside a level.
2. A normal thread that uses a whole slice (S5: 10 ms, tunable) drops one level. Time used is summed across wakeups, so short sleeps do not reset it.
3. Every boost period (one second, tunable) each normal thread returns to its `baseLevel`, which nice sets.
4. Fixed threads never move. Only root may place a thread there (X3).

There is no tick (K5): each switch sets the CPU's deadline to the earlier of the slice end and the next sleeper. A preempt count defers a switch while a spinlock is held (K2). A woken thread returns to the CPU it last ran on. From phase 4 (S6, P10) an idle CPU steals one normal thread from the busiest queue before it idles.

### Signals and job control
S7 gives interrupt, terminate, kill, child exit, stop, continue and fault. Masks are 64 bits wide so the set can grow. On return to user mode the kernel pushes a signal frame onto the user stack, the handler runs, and `signalReturn` restores the frame. An interrupted blocking call either restarts or returns `ERR_INTERRUPTED`, chosen per action.

A session holds at most one controlling terminal, and the terminal holds the foreground group, which receives interrupt and stop from the TTY layer (U6).

### Program loading
The parser is the `lib/` ELF code shared with boot. The kernel accepts a static executable for this port, refuses any segment that is both writable and executable (X6), builds the standard initial stack (arguments, environment, auxiliary vector) and enters at the ELF entry. A file starting with `#!` names an interpreter and at most one argument within its first 256 bytes; the loader restarts on the interpreter, in a loop limited to four levels (T11).

The phase 5 dynamic linker is a user program in `/lib`. From the kernel it needs `PT_INTERP` handling, auxiliary entries for the program headers and interpreter base, private file mappings at chosen addresses, and `memProtect`. From the build it needs position-independent libraries and thread-local storage for loaded modules.

### Waiting and IPC

```c
int64_t futexWait(uint32_t *address, uint32_t expected, int64_t timeoutNs);
int64_t futexWake(uint32_t *address, uint32_t count);
```

- S11: waiters hash on address space and address into kernel wait queues (K3).
- S12: a pipe is a 64 KiB ring. Local sockets bind to filesystem paths, carry descriptors as ancillary data and report peer credentials. Shared memory is an anonymous memory object (M4) behind a descriptor.
- S13: every descriptor type implements one readiness hook. `fdPoll` uses it, and so does the phase 5 event queue, itself a descriptor.

### Inspection, crashes, limits
- S14: `/proc/<pid>/` holds `status` (key=value, U13), `cmdline`, `maps`, `threads` and `fds/`.
- S15: an unhandled fault writes one report to the kernel log (K8): process, signal, fault address, registers, mapping bases, and a frame-pointer backtrace as binary-plus-offset pairs, read with the K14 copies.
- S16: limits on open descriptors and memory, inherited by children.

## Per-architecture notes
Everything goes through `ArchContextOps`. `TrapFrame` is opaque, so this design needs operations the contract lacks: push and restore a signal frame, read the user program counter and stack pointer, dump user registers, walk a user backtrace, rewind a syscall for restart, and report the port's ELF machine number and user address limit.

## Build order
- Phase 1: kernel threads, wait queues, one run queue.
- Phase 2: user processes, spawn, static ELF loader, `#!`, wait, feedback levels, nice, futex, threads, pipes, basic crash report.
- Phase 3: fork and exec, signals, groups, sessions, controlling terminals, `fdPoll`, `/proc`, limits, file-backed loading.
- Phase 4: second CPU, stealing, wake by IPI.
- Phase 5: local sockets, shared memory, event queue, dynamic linker, fixed class in use.
- Phase 6: core dumps.

## Test plan
- Host: scheduler policy on the fake port (level moves, boost, steal choice); ELF and `#!` parsing, fuzzed (Q8); signal state machine; pipe ring.
- In-kernel: thread create and exit stress; preemption under lock load; sleep accuracy.
- Userland: spawn with descriptor actions; fork, exec and wait; pipe end-of-file; each signal's default and handler; job control through a pseudo-terminal; futex ping-pong; poll timeouts; limits; `/proc`; a crash whose report the boot test reads.
- Phase 4: migrating threads, none lost. Benchmarks: context switch (Q10), slice length (S5).

## Risks and open questions
- S7 against D6, S8, U4, U6: the subset omits hangup, broken pipe, alarm, window change, quit, abort and the user signals, and does not say whether terminal read and write stops count as "stop". `abort`, `alarm` and editor resize need these.
- Roadmap against K3, K4: phase 1 has sleeping locks and kernel threads, but lists the scheduler in phase 2.
- S10 with X6: randomizing a static image needs self-relocating position-independent executables. Not decided.
- S4 with S6, S11, K3: the fixed class wins across CPUs only if a woken fixed thread is pushed to another CPU. Nothing caps a spinning fixed thread, and no lock has priority inheritance.
- K12 with S7, S8: POSIX names processes and groups by number, so pids exist beside process descriptors. Threads, futexes and mappings are not descriptors.
- S12 with U12, U17, X4: local sockets arrive in phase 5, but phase 3 services have only pipes.
- S12 with W2, X1: a client that shrinks a shared buffer can fault the compositor.
- S16 with M5, M10: "memory" could mean mapped or resident bytes. Nothing limits process count.
- S13, S14, S15: no phase is given for `poll`, `/proc` or crash reports. S15 does not say who symbolizes, or whether "system log" means K8 or U17.
- P8: the contract gaps under per-architecture notes.
