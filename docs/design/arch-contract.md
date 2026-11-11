# Arch contract

Status: draft      Implements: P8, P10, T12, M13, K5, K6, K14, H4, H14, B10

## Purpose
The arch contract is the complete list of things `kernel/` may ask of a CPU port. Each port in `arch/<port>` fills a set of ops tables and registers them before `kernelMain` runs. Nothing outside `arch/` and `boot/` includes a per-port header or calls a per-port symbol.

## Decisions applied
- P8: every operation is a function pointer in a table, the hot ones included. P8 rejected the mixed option, so per-port types are opaque and the tables report their sizes.
- P8 names five tables. This note adds four that other decisions need: early console (B10), per-CPU (P10), atomics (K3) and reset (H14).
- T12: assembly exists only behind these tables and in `archEntry`.
- M13: the page-table walker is generic. The MMU table only encodes and decodes entries.
- K5, K6: the timer table is a free-running counter plus one deadline per CPU.
- T11: the context table only saves and restores user floating-point state.

## Design

### Registration
`archEntry` (see boot.md) calls the port's C setup, which picks its tables, calls `archRegister` once, then calls `kernelMain(bootInfo)`. `archRegister` asserts that every pointer is set (K11). A port may choose between implementations by CPU feature (P4). After registration the tables never change.

```c
typedef struct ArchOps {
    const ArchEarlyConsoleOps *earlyConsole;
    const ArchCpuOps          *cpu;
    const ArchPerCpuOps       *perCpu;
    const ArchAtomicOps       *atomic;
    const ArchIrqOps          *irq;
    const ArchTimerOps        *timer;
    const ArchMmuOps          *mmu;
    const ArchContextOps      *context;
    const ArchResetOps        *reset;
} ArchOps;

void archRegister(const ArchOps *ops);
extern const ArchOps *arch;
```

Calls in the other direction also go through a table, which `kernel/` hands to the port:

```c
typedef struct KernelTrapOps {
    void (*irq)(uint32_t irq, TrapFrame *frame);
    void (*timer)(TrapFrame *frame);
    void (*ipi)(TrapFrame *frame);
    void (*pageFault)(TrapFrame *frame, uintptr_t address, uint32_t access);
    void (*syscall)(TrapFrame *frame);
    void (*exception)(TrapFrame *frame, uint32_t kind);
} KernelTrapOps;
```

### The tables

```c
typedef struct ArchEarlyConsoleOps {
    void (*init)(const BootInfo *bootInfo);
    void (*putChar)(char c);
    int  (*getChar)(void);
} ArchEarlyConsoleOps;

typedef struct ArchCpuOps {
    void   (*initCpu)(void);
    void   (*idle)(void);
    void   (*relax)(void);
    void   (*halt)(void);
    int    (*startCpu)(uint64_t cpuId, void (*entry)(void *), void *arg, void *stackTop);
    void   (*sendIpi)(uint64_t cpuId);
    size_t (*backtrace)(const TrapFrame *frame, uintptr_t *out, size_t max);
} ArchCpuOps;

typedef struct ArchPerCpuOps {
    void    (*set)(PerCpu *self);
    PerCpu *(*get)(void);
} ArchPerCpuOps;

typedef struct ArchAtomicOps {
    uint64_t (*load)(const uint64_t *p);
    void     (*store)(uint64_t *p, uint64_t value);
    uint64_t (*exchange)(uint64_t *p, uint64_t value);
    bool     (*compareExchange)(uint64_t *p, uint64_t *expected, uint64_t desired);
    uint64_t (*fetchAdd)(uint64_t *p, uint64_t delta);
} ArchAtomicOps;

typedef struct ArchIrqOps {
    void     (*init)(const BootInfo *bootInfo, const KernelTrapOps *upcalls);
    void     (*initCpu)(void);
    IrqState (*disable)(void);
    void     (*restore)(IrqState state);
    void     (*enable)(void);
    void     (*mask)(uint32_t irq);
    void     (*unmask)(uint32_t irq);
    void     (*endOfInterrupt)(uint32_t irq);
} ArchIrqOps;

typedef struct ArchTimerOps {
    void     (*init)(const BootInfo *bootInfo);
    void     (*initCpu)(void);
    uint64_t (*frequency)(void);
    uint64_t (*readCounter)(void);
    void     (*setDeadline)(uint64_t counterValue);
    void     (*cancel)(void);
} ArchTimerOps;

typedef struct ArchMmuOps {
    uint32_t  levels;
    PageEntry (*makeLeaf)(uint64_t phys, uint32_t flags, uint32_t level);
    PageEntry (*makeTable)(uint64_t tablePhys);
    uint32_t  (*decode)(PageEntry entry, uint32_t level, uint64_t *phys);
    int       (*initUserRoot)(uint64_t rootPhys);
    void      (*activate)(uint64_t rootPhys);
    void      (*flushPage)(uintptr_t virt);
    void      (*flushAll)(void);
    int       (*copyFromUser)(void *dst, uintptr_t src, size_t length);
    int       (*copyToUser)(uintptr_t dst, const void *src, size_t length);
} ArchMmuOps;

typedef struct ArchContextOps {
    size_t contextSize, trapFrameSize, userFpSize;
    void (*initKernel)(ArchContext *ctx, void *stackTop, void (*entry)(void *), void *arg);
    void (*initUser)(TrapFrame *frame, uintptr_t entry, uintptr_t stack, uintptr_t arg);
    void (*switchTo)(ArchContext *from, ArchContext *to);
    void (*enterUser)(TrapFrame *frame);
    void (*setKernelStack)(void *stackTop);
    void (*setUserTls)(uintptr_t value);
    void (*saveUserFp)(void *state);
    void (*restoreUserFp)(const void *state);
    void (*syscallArgs)(const TrapFrame *frame, uint64_t *number, uint64_t args[6]);
    void (*setReturn)(TrapFrame *frame, int64_t value);
} ArchContextOps;

typedef struct ArchResetOps {
    void (*shutdown)(void);
    void (*reboot)(void);
} ArchResetOps;
```

`TrapFrame`, `ArchContext` and the user floating-point area are opaque blocks that `kernel/` allocates at the reported size. `PageEntry` and `IrqState` are `uint64_t`. Page flags, fault access bits and exception kinds are portable constants defined in `kernel/`. `getChar` returns a negative value when nothing is waiting. `halt`, `enterUser`, `shutdown` and `reboot` never return. Errors are negative codes (K7).

## Per-architecture notes

| Table | x86_64 | riscv64 | aarch64 (phase 4) |
| --- | --- | --- | --- |
| Early console | 16550 on COM1 | SBI debug console | PL011 |
| CPU | `hlt`, `pause`, local APIC start-up and IPI | `wfi`, SBI hart start and IPI | `wfi`, PSCI CPU on, GIC software interrupt |
| Per-CPU | GS base | `tp`, swapped with `sscratch` on trap entry | `TPIDR_EL1` |
| Atomics | `lock` prefix | A extension (P3) | exclusive load and store |
| Interrupts | IDT, local and I/O APIC, flags register | `stvec`, PLIC, `sstatus.SIE` | `VBAR_EL1`, GICv3, `DAIF` |
| Timer | TSC, local APIC one-shot | `time` CSR, SBI set timer | generic timer |
| MMU | 4 levels, `CR3`, `invlpg` | `satp`, `sfence.vma` | `TTBR0_EL1`, `tlbi` |
| Context | TSS stack, FS base, FXSAVE | `sscratch`, `tp`, F and D registers | `SP_EL1`, `TPIDR_EL0`, FP and SIMD registers |
| Reset | UEFI runtime service | SBI system reset | PSCI |

`initUserRoot` copies the kernel half into a new root on x86_64 and RISC-V. It does nothing on AArch64, which has a separate kernel root register.

## Build order
- 0.4: `ArchOps`, `archRegister`, early console.
- Phase 1: CPU, per-CPU, atomics, interrupts, timer, MMU encoding, kernel half of context.
- Phase 2: user half of context, user copies, `initUserRoot`, `syscall` upcall.
- Phase 4: `startCpu`, `sendIpi`, the AArch64 port.

## Test plan
- Host: a fake port implements every table in plain C, so `kernel/` unit tests run on the host (Q1).
- In-kernel, the same suite on every port: interrupt state nests and restores, a deadline fires once and not early, the counter never goes backwards, entries round-trip through encode and decode, two kernel threads switch and keep their registers.
- Build check: CI fails if `kernel/` or `drivers/` includes anything from `arch/` or uses an architecture `#ifdef` (principle 1).
- Benchmarks (Q10): syscall and context-switch cost per port, so the price of the tables is measured.

## Risks and open questions
- P8, the cost of tables on the hottest primitives. Masking interrupts, reading the per-CPU pointer and an atomic operation are each one to three instructions when inlined. Through a table each becomes a pointer load, an indirect call and a return, blocks inlining and clobbers caller-saved registers. A spinlock acquire and release makes at least four such calls. Every lock, preemption check (K2) and reference count pays this. The custom CPU may have no indirect branch predictor, and if X10 is ever reversed, mitigated indirect calls cost more again. Question for Graeson: should P8 allow a small inline carve-out, one per-port header limited to interrupt masking, the per-CPU pointer and atomics, with everything else staying in tables? Atomics could instead come from C17 `<stdatomic.h>` with no port code. Recommendation: allow it, as an approved change to P8. Until then all three stay in tables.
- P8: the decision does not say whether calls from the port into `kernel/` are part of the contract. This note uses `KernelTrapOps`; direct calls would be simpler.
- H4, P7: interrupt controllers sit behind `ArchIrqOps` here, but the PLIC and GIC are found by device tree like drivers. Where controller code lives is unsettled.
- M13, P3: M13 assumes one tree shape. Sv39 has three levels and AArch64 splits the root, so `levels` and `initUserRoot` exist. The RISC-V paging mode is undecided.
- K6, P4: an invariant TSC is not part of baseline x86_64, and calibration needs a reference clock that no decision names.
- P7, H4, roadmap: ACPI and device tree parsing are in phase 4, but phase 1 interrupts and timers need controller addresses and the timer frequency.
- T1, T12: the extension list and the assembly list are not written yet.
- P3, P4: there is no baseline decision for AArch64.
