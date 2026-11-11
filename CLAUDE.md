# yart

A from-scratch, multi-architecture operating system: kernel, command line, GUI. Written in C with clang. Runs on x86_64 and RISC-V 64 from the first commit, AArch64 from phase 4, and later on a custom CPU. Owner: Graeson. Almost all code is written by Claude Code agents from approved specs (D10).

**Status and next step are always in `docs/status.md`. Read it first.**

## Read before working

1. `docs/status.md`: where the project is, what is taken, what is next.
2. `docs/roadmap.md`: phases and slices.
3. `docs/principles.md`: tie-breakers.
4. `docs/architecture.md`: layers, repo layout, the rules that hold it together.
5. `docs/workflow.md`: the agent pipeline in full.
6. For the slice in hand: its spec in `docs/specs/`, the design notes in `docs/design/`, and the decisions it cites in `docs/decisions/`.

## Non-negotiables

- **No approved spec, no code.** A spec's `Status:` must read `approved`, and only Graeson sets that.
- **Decisions are settled.** `docs/decisions/` holds 248 answered design questions. Cite them by ID. Never contradict one in code, and never edit one inside a slice. If one looks wrong, write it into `docs/open-questions.md` and stop if it blocks you.
- **Portable core.** No architecture-specific code or `#ifdef` outside `arch/` and `boot/`. The kernel reaches the CPU only through the arch ops tables (P8).
- **Ours from the ground up.** No third-party code in the tree, including bootloaders and firmware. The only exceptions are the ones listed in D7. Do not copy from other kernels or tutorials.
- **Every feature ships with tests**, on every required port. Never weaken, skip or delete a test to get to green.
- **Main always boots.** Work on a branch, land by pull request.
- **Stopping is allowed.** If you are blocked or a loop limit is hit, write down what you tried and stop. A plain "could not finish, here is why" beats a faked result.

## How work happens: use the agents

Do not build a slice in one long solo pass. Run the pipeline with `/slice <id>`; it delegates to the agents in `.claude/agents/`:

| Step | Agent | Gate |
| --- | --- | --- |
| Write the spec | `spec-writer` | Graeson approves the spec |
| Confirm hardware and standards facts | `spec-researcher` | |
| Write failing tests from the spec | `test-writer` | |
| Build until tests pass | `implementer` | |
| Diagnose any failure | `boot-debugger` | |
| Review with fresh context | `reviewer` and `port-checker`, in parallel | |
| Explain it and update design notes | `walkthrough-writer` | Graeson reads and merges |

Other commands: `/spec <id>` drafts a spec only, `/phase-plan <n>` cuts a phase into slices, `/sweep <n>` runs the end-of-phase bug hunt with `sweeper` agents in parallel, `/status` reports where things stand.

Keep the roles separate. The test writer does not read the implementation. The implementer does not edit tests. The reviewer and port checker do not edit anything.

## Commands

The `./x` wrapper arrives in slice 0.2. Until then there is nothing to build.

```
./x build <port>        x86_64 | riscv64 | aarch64
./x run <port>          boot in QEMU
./x test [host|<port>|all]
./x image <port>        build a bootable disk image
./x fmt   ./x lint   ./x config
```

## Style

- C17, warnings are errors, clang-format and clang-tidy clean (T1, T8, T10).
- camelCase functions and variables, PascalCase types, `UPPER_SNAKE` constants. POSIX-layer symbols keep their standard names (T7).
- In the kernel: no floating point, no variable-length arrays, no unbounded recursion, no allocation in interrupt handlers (T11). Assembly only for the short allowed list (T12).
- Commit messages cite the slice and the decision IDs they rely on.

## Ending a session

Update `docs/status.md` in the same pull request: slice state, anything left undone, anything the next session must know. Sessions do not remember each other; the repo is the memory.
