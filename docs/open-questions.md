# Open questions

Things only Graeson can settle, plus readings Claude took of his answers that he should confirm. Remove an item when it is answered and record the answer in `docs/decisions/`.

## Readings to confirm

| ID | His answer | Reading taken |
| --- | --- | --- |
| D1 | "fresh repo, no new" | Fresh repo, nothing carried over from bongOS or NexOS |
| D7 | "none at all" | No third-party code including boot and firmware, but the exceptions he approved elsewhere stand: math library (A10), ACPI interpreter (H5), cryptography (X8), text shaping (W15), phase 6 ports |
| D6 | "Own API with a POSIX layer on top, hopefully automatic translation with compiling source C code" | Native API is the real interface; POSIX C source recompiles unchanged against the POSIX layer. This is source compatibility, not binary |
| U2 with U1, U5 | "structure", and yes to POSIX `sh` | One shell: POSIX grammar for scripts, structured records between native tools, text as the fallback |
| D2, D9 | "after phase 6, with booting to virtual box"; "add virtualbox for phase 5" | VirtualBox boot is required at the end of phase 5; 1.0 is the end of phase 6 |
| K1 | "yes modules" | Module interface from phase 2, runtime loading in phase 4 |
| M9 | "yes swap" | Swap in phase 6 |

## Needs an answer

| # | Question | Why it matters | Needed by |
| --- | --- | --- | --- |
| 1 | P8 puts the whole arch contract behind function-pointer tables. Should the hottest primitives (interrupt masking, per-CPU pointer, atomics) be allowed as inline functions instead? | An indirect call on every lock and unlock is measurable. A small carve-out keeps the tables for everything else | Slice 0.4 |
| 2 | T14: where does the Kconfig front end come from? An existing Python implementation used as a host tool (like clang is), or our own? | D7 covers code in the tree; host tools are a grey area | Slice 0.2 |
| 3 | Q15: what is the monthly budget for cloud sessions and CI? | Sets how many agents run in parallel and how big slices can be | Before phase 1 |
| 4 | W17: the design system draft needs your eye. "Like KDE Plasma" is the direction; the tokens are ours | The look is taste, not engineering | Before phase 5 |
| 5 | G3 and G7: the browser and Linux binary compatibility are both wanted after 1.0. Which comes first, and is the browser a port of an existing engine or our own? | They pull the post-1.0 kernel work in different directions | After 1.0 |
| 6 | B3: confirm the AArch64 boot chain (our UEFI loader rebuilt, or a firmware-free path) | Decided when the port starts | Phase 4 |
| 7 | P14: which real RISC-V or ARM board, if any | Hardware purchase | Phase 6 |

## Honest sizing notes

- **A real web browser (G3)** is the largest single item on the wish list, larger than the kernel. It is listed after 1.0 for that reason.
- **Linux binary compatibility (G7)** means implementing a large part of Linux's syscall behavior exactly. The personality hook keeps the door open; walking through it is a project of its own.
- **Our own bootloaders and firmware (B1, B2, P6)** add real work to phase 0, and they are the hardest code to debug. They are also what the custom CPU will need, so the work is not wasted.
- **Our own filesystem from phase 2 (F1)** means writing the host tools that create and check images, since nothing else understands the format.
