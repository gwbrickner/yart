---
name: spec
description: Draft or revise the spec for one yart slice without building it.
argument-hint: [slice-id]
disable-model-invocation: true
---

Draft or revise the spec for slice $ARGUMENTS.

1. Delegate to `spec-writer` with the slice id.
2. Send any `VERIFY:` items to `spec-researcher` and have the answers added to the spec's References.
3. Update the slice's state in `docs/status.md` to `spec draft`.
4. Report the spec path and its open questions, then stop. Graeson approves.
