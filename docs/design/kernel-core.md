# Kernel core

Status: draft      Implements: K1 to K14

## Purpose
Kernel core is what the rest of `kernel/` and `drivers/` stands on: locks, interrupts, timers, errors, logging, panic, the debug shell, user-memory copies and modules. It sits directly above the arch ops tables (P8, arch-contract.md).

## Decisions applied
- K2, P10: kernel code can be preempted unless it holds a spinlock, has interrupts masked or has raised its preemption count.
- K3: spinlocks always mask interrupts: one kind, no variant to pick wrongly (principle 4).
- K7, A13: `lib/errors.def` generates the kernel's list and libc's `errno.h`, so constants keep POSIX names (`-ENOENT`). Functions return `int`; results use out parameters.
- K11: one `ASSERT`, always compiled. Expensive checks are Kconfig debug features (K3, M16).
- K13: every interface below carries its contract as a header comment.

## Design

### Preemption and locks
```c
void preemptDisable(void);
void preemptEnable(void);
IrqState spinLock(SpinLock *lock);
void     spinUnlock(SpinLock *lock, IrqState state);
int  waitSleep(WaitQueue *queue, SpinLock *lock, IrqState *state, uint64_t timeoutNs);
void waitWake(WaitQueue *queue, uint32_t count);
void mutexLock(Mutex *mutex);
void mutexUnlock(Mutex *mutex);
```
The preemption count lives in `Thread`. Interrupt upcalls reschedule before returning when it is zero and a reschedule is pending. `waitSleep` releases `lock`, sleeps and retakes it as one step, so no wake-up is lost.

Lock-order checker (`CONFIG_DEBUG_LOCKS`): every lock names a static `LockClass`, and each thread keeps a fixed-depth stack of held locks. Taking B while holding A records the edge A to B. A new edge that closes a cycle, found by an iterative search (T11), panics and names both sites. So does sleeping with a spinlock held.

### Interrupts and deferred work (K4)
```c
typedef enum IrqResult { IRQ_NONE, IRQ_HANDLED, IRQ_DEFERRED } IrqResult;
int  irqRegister(uint32_t irq, IrqResult (*handler)(uint32_t irq, void *arg), void *arg);
void irqDone(uint32_t irq);
void workQueue(WorkItem *item);
```
A handler acknowledges its device and never sleeps or allocates (T11). One that returns `IRQ_DEFERRED` has queued a `WorkItem`; the line stays masked until that work calls `irqDone`. Work runs in one worker kernel thread per CPU.

### Time (K5, K6)
```c
uint64_t timeMonotonicNs(void);
int64_t  timeWallNs(void);
void timerStart(Timer *timer, uint64_t deadlineNs);
bool timerCancel(Timer *timer);
```
Each CPU keeps a deadline-sorted list of caller-embedded `Timer` records. The head programs the port's deadline; an empty list cancels it, so an idle CPU takes no interrupts. Ticks become nanoseconds through a multiplier and shift computed once from the counter frequency, without `__int128` (T1). Wall time is the real-time clock value read at boot plus monotonic time, corrected by network sync in phase 6.

### Log (K8)
```c
void    logWrite(LogLevel level, const char *format, ...);
ssize_t logRead(uint64_t *sequence, void *buffer, size_t size);
```
Records (sequence, time, level, CPU, text) go into a static byte ring under one spinlock, so logging works before the heap exists. Records at or above `loglevel=` (B9) are mirrored to serial. `/dev/klog` gives `dmesg` the ring (K12).

### Panic, assertions and symbols (K9, K11)
```c
_Noreturn void panic(const char *format, ...);
#define ASSERT(condition)
typedef struct KernelSymbol { uint32_t imageOffset, nameOffset; } KernelSymbol;
int symbolLookup(uintptr_t address, const char **name, uintptr_t *offset);
```
`panic` masks interrupts, claims an atomic flag so a nested panic halts at once, stops other CPUs by IPI (phase 4), then writes to the early console, bypassing the log lock: `@yart-panic`, message, register dump and symbolized backtrace. Then it halts. A failed `ASSERT` panics with file and line.

Embedding the symbol table: the build links `kernel.elf`, a host tool writes its function symbols as a sorted `KernelSymbol` array plus a name blob, and the kernel is linked again with that object. The section sits after every other loaded section, so adding it moves no symbol; the tool fails the build if any address changed. Offsets are relative to the image base, so the table survives load-address randomization (B11). Loaded modules register their own tables.

### Debug shell (K10)
A kernel thread reads lines from the early console. `DEBUG_COMMAND(name, help, run)` adds a command to a linker-section array: `threads`, `mem`, `locks`, `log`, `timers`, `read`. `read` asks the page-table walker (memory.md) before touching an address.

### User memory (K14)
```c
typedef struct UserPtr { uintptr_t address; } UserPtr;
int copyFromUser(void *dst, UserPtr src, size_t length);
int copyToUser(UserPtr dst, const void *src, size_t length);
```
C cannot dereference a `UserPtr`, so the compiler enforces K14. The wrappers check the range against the user address limit and call the port's user copies. A fault inside a copy first takes the normal page-fault path; only an unresolved fault takes the recovery path and returns `-EFAULT`.

### Modules (K1)
```c
typedef struct Module { const char *name; uint64_t kernelBuildId; int (*init)(void); void (*exit)(void); } Module;
#define MODULE_DEFINE(name, init, exit)
#define EXPORT_SYMBOL(symbol)
int moduleLoad(File *file);
int moduleUnload(const char *name);
```
Phase 2: `MODULE_DEFINE` emits a `Module` into a linker section and boot calls each `init`, where drivers register their match tables (H1). `EXPORT_SYMBOL` adds a name and address to the export table. Phase 4: the same source is built as a relocatable ELF. `moduleLoad` parses it with the `lib/` ELF code, rejects a different `kernelBuildId`, takes memory from the module area (memory.md), copies sections, resolves undefined symbols against the sorted export table, applies relocations through the port, sets final page permissions and calls `init`.

## Per-architecture notes
Used from arch-contract.md: the interrupt, per-CPU, atomic, timer and early console tables, the user copies, `backtrace`, `sendIpi` and `switchTo`.

Needed and not yet in that contract:
- an op that applies one relocation record (K1);
- an op that prints an opaque `TrapFrame` (K9);
- a way for `pageFault` to report "unresolved", so the port can run its copy recovery (K14);
- `backtrace` with no frame, meaning "from my caller";
- upcalls running on the interrupted thread's kernel stack (K2).

Module reach: within 2 GiB of the kernel image on x86_64 and riscv64, 128 MiB on aarch64. User-access guards: SMAP where present, `sstatus.SUM` clear, PAN.

## Build order
Later slices are not cut yet. Suggested order:
- 0.4: minimal `logWrite`, `panic`, `ASSERT`.
- Phase 1: error list, spinlocks and checker, interrupts, timers, log ring, symbol table, kernel threads, wait queues, mutexes, work queues, debug shell.
- Phase 2: user copies, `/dev/klog`, module interface with drivers linked in.
- Phase 4: IPI stop on panic, runtime module loading.

## Test plan
- Host: timer ordering, tick conversion, log ring wrap, cycle detection, symbol lookup, module ELF fuzzing (Q8).
- In-kernel: lock inversion caught, timers never fire early, a bad user address returns `-EFAULT`.
- Boot: a forced panic prints `@yart-panic` and the right function name; a scripted debug shell; a test module loads (phase 4).

## Risks and open questions
- K3, K4, S3, roadmap: mutexes, wait queues and deferred work need kernel threads in phase 1, but the scheduler is a phase 2 item. Phase 1 needs a placeholder.
- K2, P8: reading the current thread through the per-CPU table is only safe with interrupts masked, so each preemption-count change costs three indirect calls (open question 1).
- K1, K9, K14, P8: the contract gaps listed above.
- K14, P4: SMAP is not baseline x86_64. Without it only `UserPtr` guards direct access.
- K6, B5, P7: phase 1 has no real-time clock source: drivers arrive in phase 2, discovery in phase 4, and `BootInfo` carries no time.
- K1, B11: boot.md says the shared ELF code does no relocation until phase 6; modules need it in phase 4. No decision covers a stable module ABI.
- K9, K10: whether a panicked kernel may enter the debug shell before halting is unsettled; this note halts.
- K10, B10, X1: from phase 2 the unauthenticated debug shell shares the serial line with the user console. An escape sequence and a release default are undecided.
- K7, D6: whether libyart shows POSIX error names or its own is unsettled.
