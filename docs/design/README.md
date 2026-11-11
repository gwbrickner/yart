# Design notes

One note per subsystem. A design note turns the decisions for its area into a buildable design: structures, interfaces, build order and tests. Specs in `docs/specs/` are cut from these notes one slice at a time.

A note never overrides `docs/decisions/`. If the two disagree, the note is wrong, or the decision needs changing through the process in `docs/decisions/README.md`.

## Template

```
# <Subsystem>

Status: draft | reviewed | current      Implements: <decision IDs>

## Purpose
Two or three sentences: what this subsystem is for and where it sits.

## Decisions applied
Only the decisions that need spelling out, as "ID: what it means here".

## Design
Structure, key data structures, and interfaces sketched as C declarations
(camelCase, T7). Interface sketches only, no implementations.

## Per-architecture notes
What each port must provide. Omit if the subsystem is architecture-neutral.

## Build order
Which phase and slice introduces which part.

## Test plan
Host unit tests, in-kernel self tests, userland tests, boot tests (Q1).

## Risks and open questions
What the decisions leave unsettled, and any conflicts found between decisions.
```

Keep a note between 700 and 1300 words. Longer means it should be split.
