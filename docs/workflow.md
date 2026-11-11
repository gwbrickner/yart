# Workflow

How a piece of yart goes from an idea to merged code. Claude Code agents do the work; Graeson approves at two gates. This file is the long form of the summary in `CLAUDE.md`.

## Units of work

- **Phase**: one row of `docs/roadmap.md`. Closes with a sweep.
- **Slice**: one spec, one branch, one reviewable diff, one pull request (Q5). Named `<phase>.<n>`, for example `0.5`.
- **Spec**: `docs/specs/phase-<n>/<slice>-<name>.md`, written from the template. Nothing gets built without an approved spec (Q4).

## The agents

Each agent is defined in `.claude/agents/`. The main session orchestrates; agents do not hand work to each other directly.

| Agent | Job | Writes to | Never does |
| --- | --- | --- | --- |
| `spec-writer` | Turns a roadmap slice into a spec, citing decisions and design notes | `docs/specs/` | Write code |
| `spec-researcher` | Looks up hardware and standards facts (UEFI, RISC-V, virtio, ELF) with sources | The spec's References section, through the orchestrator | Guess; cite from memory |
| `test-writer` | Writes the tests the spec lists, before the code exists | `tests/` | Read or edit the implementation |
| `implementer` | Builds the slice until the tests pass | Source directories | Edit tests or decisions |
| `boot-debugger` | Finds the root cause when a build, boot or test fails | Nothing; reports | Patch the code |
| `reviewer` | Reviews the diff against the spec, decisions and principles with fresh context | Nothing; reports | Edit anything |
| `port-checker` | Checks the arch boundary and that every required port builds and passes | Nothing; reports | Edit anything |
| `walkthrough-writer` | Explains the merged slice for Graeson; keeps design notes current | `docs/walkthroughs/`, `docs/design/` | Change behavior |
| `sweeper` | Hunts bugs in one subsystem at the end of a phase | `tests/` (regression tests), reports | Fix what it finds |

Why the split: the test writer has not seen the code, so tests check the spec and not the implementation. The reviewer has not seen the implementer's reasoning, so it reads the diff the way Graeson will (Q6).

## The slice pipeline

Run with `/slice <id>`.

```
 spec-writer ──► GATE 1: Graeson approves the spec
      │
      ▼
 spec-researcher (only when the slice touches hardware or a standard; several may run in parallel)
      │
      ▼
 test-writer ──► tests exist and fail for the right reason
      │
      ▼
 implementer ──► ./x test all
      │              │ fails
      │              ▼
      │         boot-debugger ──► implementer   (at most 3 rounds, then stop and report)
      ▼
 reviewer  +  port-checker        (in parallel, both read-only)
      │ must-fix findings ──► implementer        (at most 2 rounds, then stop and report)
      ▼
 walkthrough-writer
      │
      ▼
 pull request ──► GATE 2: Graeson reads and merges
```

Rules the orchestrator enforces:

1. No approved spec, no build. A spec's `Status:` line must read `approved`.
2. Tests come first and belong to the test writer. If the implementer thinks a test is wrong, it says so and the orchestrator sends the question back to the test writer with the spec.
3. A stop is a result. When a loop limit is hit, write what was tried into the pull request or `docs/status.md` and stop. Do not lower the bar to get to green.
4. Nobody edits `docs/decisions/` inside a slice. A decision that looks wrong goes to `docs/open-questions.md` and the slice stops if it is blocked by it.
5. Every pull request description lists: the spec, the decision IDs it relies on, test results per port, and the reviewer's and port checker's verdicts.

## Starting and ending a phase

- **Start** with `/phase-plan <n>`: the spec writer cuts the phase into slices and drafts every spec. Graeson approves the slice list before any spec is built. Independent slices may then run in separate sessions.
- **End** with `/sweep <n>` (Q7): one sweeper per subsystem, in parallel, each reporting bugs with a reproduction. Findings become regression tests and fix slices. The phase closes when the checklist in Q12 passes: milestone works, tests pass on every required port, design notes are current, sweep is done.

## Parallel work

- Read-only agents (reviewer, port checker, sweepers, researchers) run in parallel freely.
- Two writing agents never touch the same directory at the same time. The test writer and implementer run in sequence for that reason.
- Slices that do not depend on each other can run in separate sessions on separate branches. `docs/status.md` is the lock: a slice marked `building` is taken.

## Session protocol

Cloud sessions start with no memory of earlier ones. The repo is the memory.

- **Start**: read `CLAUDE.md`, then `docs/status.md`. Run `/status` if unsure what is next.
- **During**: one slice per session unless told otherwise.
- **End**: update `docs/status.md` (slice state, anything left undone, anything a future session must know) in the same pull request.

## Changing the plan

Design notes and specs change freely through pull requests. Decisions change only with Graeson's approval and a line in `docs/decisions/CHANGELOG.md` (Q13). This workflow changes the same way a design note does.
