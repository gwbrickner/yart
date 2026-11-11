# <slice id>: <name>

Status: draft | blocked | approved        Phase: <n>        Depends on: <slice ids>
Decisions: <IDs this slice relies on>     Design notes: <files>

## Goal

One paragraph. What exists after this slice that did not exist before, and why it is the next thing to build.

## Interfaces

Exact files, function signatures (camelCase), structs and constants this slice adds or changes. The implementer should not have to choose a name.

## Behavior

Numbered statements of what the code must do, each one testable. Include error cases and limits. Cite the decision ID behind each design choice.

## Per-port requirements

What each required port must provide for this slice. "None" if the slice is architecture-neutral.

## Tests

One row per test. The test writer builds these without seeing the implementation.

| # | Kind | Ports | What it checks | Expected result |
| --- | --- | --- | --- | --- |

Kinds: host, kernel, userland, boot (Q1).

## Done when

A checklist. Every item must be true before review.

- [ ] All tests above pass on every listed port
- [ ] `./x lint` is clean
- [ ] ...

## Out of scope

What this slice must not touch, especially the tempting neighbours.

## References

Hardware and standards facts the slice depends on, each with its source. Mark anything unchecked with `VERIFY:`.

## Open questions

Anything no decision or design note settles. If one blocks the slice, set the status to `blocked`.
