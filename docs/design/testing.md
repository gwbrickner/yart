# Testing

Status: draft      Implements: Q1 to Q12

## Purpose
Every slice arrives with tests written from its spec, and CI proves each push on every required port. This note defines the test kinds, what CI runs and what closes a phase.

## Decisions applied
- Q1: pure logic lives in libraries (`lib/`, `fs/yartfs/`) that also build on the host.
- Q2, P13: every push builds and boots x86_64 and riscv64, AArch64 from phase 4. Host tests run once; the other three kinds run per port (reading to confirm).
- T9, D7: host tests run under address and undefined-behavior checks; test kernels are debug kernels. The harness is ours.

## Design

### The four kinds
- Host unit tests, `tests/host/`: built and run natively. For allocators, parsers, yartfs, host tools.
- In-kernel self tests, `tests/kernel/`: linked into a test kernel, run at boot. For code that needs real page tables, interrupts and locks.
- Userland test programs, `tests/user/`: packed into the test image. For syscalls, libyart, libc, utilities.
- Boot tests, `tests/boot/`: a host runner drives QEMU over serial. They check the whole system and carry the two kinds above.

One assertion header in `lib/test/` serves host, kernel and userland:

```c
#define TEST(suite, name)
#define EXPECT_EQ(actual, expected)
void testReport(const char *path, bool passed, const char *detail);
```

In the kernel and userland, `testReport` prints protocol lines to the console:

```
@yart-test ok kernel/pmm/allocFree
@yart-test fail kernel/pmm/doubleFree expected -22 got 0
@yart-test done passed=41 failed=1
```

The panic handler prints `@yart-panic` first.

### Writing a boot test
A boot test is one declarative file.

```
# tests/boot/self-tests.boot
ports    x86_64 riscv64
config   test
memory   64M
cmdline  selftest=all
timeout  120
expect   @yart-test done passed=\d+ failed=0
```

Other verbs: `disk`, `smp`, `send <text>`, `reject <regex>`. `send` types on serial, scripting the shell and debug shell (K10).

The runner in `tools/boot-test/` starts QEMU with no display, serial on a pipe and reboot disabled, then reads lines.

- Pass: every `expect` matched in order, with no failure.
- Fail: a `@yart-test fail` line, `@yart-panic`, a `reject` match, QEMU exiting early, or the timeout.
- Timeout: required; wall-clock seconds from QEMU start, times a per-port factor. On expiry the runner prints the last 50 serial lines.
- The runner always kills QEMU itself, so no test depends on guest power-off.

`memory` defaults to 64M, or 256M for desktop tests, which keeps the P11 minimums tested. Tests use software emulation only, so machines agree.

### Directory layout
```
tests/host/         mirrors the source tree
tests/host/fuzz/    per parser: harness, corpus, regress/
tests/kernel/       mirrors kernel/ and drivers/
tests/user/         programs; sh/ (U5 list); gui/ from phase 5
tests/boot/         *.boot files; drivers/ (H17), perf/, frames/
```

### CI on every push (Q2)
Inside the pinned container:

1. `./x lint`.
2. `./x test host`: host unit tests and every saved fuzz input.
3. Per required port, `./x test <port>`: build, then every boot file, which covers self tests and userland programs.
4. Boot time recorded per port (B13).

`./x test all` is steps 2 and 3, so a local run equals CI. Nothing retries: an intermittent failure is a bug.

### Merge rule (Q3, Q6, Q11)
A pull request merges only when:

- CI is green on every required port.
- Every row of the spec's test table exists as a test, and a bug fix carries a test that failed before it. CI rejects a pull request that changes source and nothing under `tests/`.
- No test was weakened, skipped or deleted; a port-specific skip is named in the spec.
- A fresh review session, then Graeson, approved it.

### Fuzzing (Q8)
Each parser is a host library with a harness: one function taking bytes, built with clang's fuzzing engine and the sanitizers (availability to verify in slice 0.3). A harness arrives with its parser: ELF, tar, command line and device tree in phase 0, since the loaders read them; yartfs images in phase 2; fonts and PNG in phase 5.

`./x fuzz <harness> [seconds]` runs one. A crashing input is committed to `regress/` and replayed in every host run. Syscall fuzzing, from phase 3, is a userland program making seeded random calls inside a boot test.

### GUI testing (Q9, W29)
- Headless toolkit tests: the toolkit renders into a memory buffer with no compositor. Tests inject input events and assert on the widget tree (W25 roles and labels) and layout rectangles.
- Frame comparison: a boot test starts the compositor with a fixed scene and font, takes a frame through the screenshot call, and compares it in the guest with a reference from `tests/boot/frames/`. References change only in a reviewed diff.

### Performance numbers (Q10, B13)
- Boot time: QEMU start to a `@yart-boot ready` line.
- Syscall cost: a do-nothing syscall round trip, median nanoseconds. From phase 2.
- Context-switch cost: two threads passing a token through a pipe. From phase 2.
- Frame time: compositor time to compose a standard scene. From phase 5.

`./x bench <port>` prints one line per number. At each release (G8) they are appended to `tests/boot/perf/history.csv`. CI measures boot time on every push and fails past a threshold over the last recorded value.

### Phase-done checklist (Q12)
Kept as `docs/specs/phase-<n>/done.md`. The first four items are Q12.

1. The roadmap milestone is shown by a named boot test.
2. `./x test all` is green on every required port.
3. Design notes for touched subsystems read "current".
4. Sweep complete (Q7); each confirmed bug has a regression test.
5. Walkthrough notes written (Q14).
6. Performance numbers recorded (Q10).

### Test kinds by end of phase

| Phase | Host unit | Kernel self | Userland | Boot |
| --- | --- | --- | --- | --- |
| 0 | harness, loader parsers | none | none | banner on both ports |
| 1 | allocators, heap, logging | memory, interrupts, timers, locks | none | self tests, panic, debug shell |
| 2 | VFS, yartfs | scheduler, syscalls, modules | first programs | drivers, boot from yartfs |
| 3 | libc functions, shell parser | TTY, signals | libyart, libc, utilities, `sh` list | scripted shell session |
| 4 | ACPI tables | multicore, module loading | same suite on AArch64 | three ports, several cores |
| 5 | rasterizer, fonts, PNG, layout | shared memory, sockets | headless toolkit | frame comparison |
| 6 | network stack, FAT, packages | network, USB, swap | network tools | network against host (N8), installer |

## Per-architecture notes
Each port provides an early serial console (B10), a QEMU launch description in `arch/<port>/` shared by `./x run` and the runner (P12), and a timeout factor.

## Build order
- 0.3: `lib/test/`, host tests, sanitizers. 0.5, 0.6: loader harnesses. 0.7: runner, protocol, boot tests and boot time in CI.
- Phase 1: self-test registration. Phase 2: userland test runner started through `init=` (B9), driver tests, benchmarks. Phase 3: syscall fuzzer. Phase 5: headless backend, frames.

## Test plan
- Host: the runner is fed canned serial transcripts (pass, failure, panic, early exit, silence) and must give the right verdict.
- One canary per kind: a test built to fail, which CI checks is reported as failing.

## Risks and open questions
- Q2: the per-port reading above needs confirming; in phase 0 only two kinds exist.
- Q10, B13: Q10 records per release, B13 tracks in CI; no threshold is set. Emulated timing on shared machines is noisy; QEMU's counted virtual time (to verify) and one named machine for release numbers are proposed.
- Q9, W13: frames may differ between ports if the rasterizer uses floating point; exact match, tolerance or per-port references is undecided.
- D9, R8: VirtualBox, VMware, Hyper-V and real hardware cannot run in the CI container, so those boots are manual checklist items for now.
- H17, H12, H15: a real network card or laptop feature may have no QEMU model, so H17 cannot hold for its driver.
- Q15, Q11: the budget is open and the CI provider unnamed; they set fuzzing time, scheduled runs and how the merge rule is enforced.
- B9: `selftest=` adds a command line key.
- Q8: the parser list is read as examples; tar, the command line, device tree and ACPI tables are fuzzed too.
- T9, D4: userland address checks and the runner's language are open; see build-system.md.
