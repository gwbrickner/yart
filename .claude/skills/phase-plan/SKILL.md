---
name: phase-plan
description: Cut a yart phase into slices and draft every spec, at the start of the phase.
argument-hint: [phase-number]
disable-model-invocation: true
---

Plan phase $ARGUMENTS.

1. Confirm the previous phase is closed in `docs/status.md` (sweep done, Q12 checklist passed). If not, stop and say what is missing.
2. Read the phase's row in `docs/roadmap.md`, every design note it touches and the decisions they cite. If a design note is missing or still `draft` for an area this phase builds, have it written or reviewed first.
3. Delegate to `spec-writer`: cut the phase into slices, each one reviewable diff, with dependencies.
4. Add the slice table to `docs/roadmap.md` and `docs/status.md`. Stop and ask Graeson to approve the slice list.
5. After approval, delegate one `spec-writer` per slice (in parallel where slices are independent) to draft the specs. Send `VERIFY:` items to `spec-researcher`.
6. Delegate a `reviewer` to check the set of specs against the decisions and against each other for gaps and overlaps.
7. Report: slices, open questions, anything blocked. Graeson approves specs one by one.
