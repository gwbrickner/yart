---
name: slice
description: Run the full agent pipeline for one yart slice, from spec to pull request.
argument-hint: [slice-id]
disable-model-invocation: true
---

Run the slice pipeline for slice $ARGUMENTS. You are the orchestrator: you delegate every step to the named agent and you do not write the spec, tests or implementation yourself. The full rules are in `docs/workflow.md`.

1. **Preflight.** Read `docs/status.md` and the slice's row in `docs/roadmap.md`. Confirm its dependencies are merged. If the slice is marked `building`, stop: another session has it.
2. **Spec.** If no spec exists under `docs/specs/`, delegate to `spec-writer`. Then stop and ask Graeson to approve it. Continue only when the spec's `Status:` reads `approved`. If it reads `blocked`, report the open questions and stop.
3. **Facts.** If the spec has `VERIFY:` items or touches hardware, firmware or a standard, delegate each question to `spec-researcher` (in parallel when there are several) and have the answers added to the spec's References.
4. **Mark it taken.** Create a branch `slice/<id>-<name>` and set the slice to `building` in `docs/status.md`.
5. **Tests.** Delegate to `test-writer`. Confirm the tests build and fail because the feature is missing.
6. **Build.** Delegate to `implementer`. When it reports, run `./x test all` yourself.
7. **If anything fails,** delegate to `boot-debugger` for the root cause, then back to `implementer` with the diagnosis. At most three rounds. If still failing, write what was tried into `docs/status.md`, push the branch, and stop.
8. **Review.** Delegate to `reviewer` and `port-checker` in parallel. Send must-fix findings back to `implementer`, then re-run both checks. At most two rounds, then stop and report as in step 7.
9. **Explain.** Delegate to `walkthrough-writer`.
10. **Pull request.** Update `docs/status.md` to `in review`. Open a pull request whose description lists the spec, the decision IDs relied on, test results per port, and both verdicts with any should-fix notes. Stop: Graeson reads and merges.

Never edit `docs/decisions/`. Never mark a spec approved. Never weaken a test to pass.
