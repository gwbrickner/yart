# Status

Updated at the end of every session. This file is the project's memory between sessions.

**Phase:** 0 (Foundations), not started
**Last updated:** 2026-11-10

## Planning

| Item | State |
| --- | --- |
| Decisions (`docs/decisions/`) | Done: 248 settled |
| Roadmap, principles, architecture, workflow | Done |
| Agents and commands (`.claude/`) | Done |
| Design notes (`docs/design/`) | Not written yet; see the list below |
| Phase 0 specs (`docs/specs/phase-0/`) | Not written yet |
| Independent review of all planning docs | Not done yet |

Design notes still to write, each from the template in `docs/design/README.md`: `boot.md`, `arch-contract.md`, `kernel-core.md`, `memory.md`, `processes.md`, `syscalls-and-libraries.md`, `storage.md`, `drivers.md`, `shell-and-userland.md`, `security.md`, `networking-and-ports.md`, `compositor.md`, `graphics-and-text.md`, `toolkit-and-design-system.md`, `build-system.md`, `testing.md`.

## Phase 0 slices

| Slice | Name | State |
| --- | --- | --- |
| 0.1 | Repo skeleton and toolchain | spec not written |
| 0.2 | Build system | spec not written |
| 0.3 | Host test harness | spec not written |
| 0.4 | Kernel entry and boot info | spec not written |
| 0.5 | UEFI loader | spec not written |
| 0.6 | RISC-V firmware | spec not written |
| 0.7 | Boot tests in CI | spec not written |

Slice states: `spec not written`, `spec draft`, `spec approved`, `building`, `in review`, `merged`.

## Notes for the next session

- Write the design notes first, one agent per area in parallel, then the phase 0 specs, then run a fresh-context review of everything against `docs/decisions/`.
- Open questions that need Graeson are in `docs/open-questions.md`.
