---
name: sweep
description: Run the end-of-phase bug sweep across every yart subsystem built so far.
argument-hint: [phase-number]
disable-model-invocation: true
---

Run the end-of-phase sweep for phase $ARGUMENTS (Q7).

1. List every subsystem that has code, from `docs/status.md` and the source tree.
2. Delegate one `sweeper` agent per subsystem, all in parallel. Give each its subsystem's directories, design note and specs.
3. Collect the reports. For every confirmed bug, check that its regression test fails on `main`.
4. Write `docs/sweeps/phase-$ARGUMENTS.md`: confirmed bugs by severity, unconfirmed suspicions, areas not covered.
5. Group the confirmed bugs into fix slices and delegate the specs to `spec-writer`.
6. Report the summary and stop. Graeson approves the fix slices. The phase closes only when the Q12 checklist passes.

Sweepers find and prove; they do not fix.
