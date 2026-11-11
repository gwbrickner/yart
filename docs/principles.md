# Principles

When two good options conflict and no decision settles it, these break the tie. Listed in priority order.

1. **Portable core.** Nothing outside `arch/` and `boot/` knows which CPU it runs on. If a change needs `#ifdef` on an architecture in `kernel/`, the arch contract is missing something; fix the contract.
2. **Every feature ships with a test.** No test, no merge (Q3). A bug fix starts with a test that fails.
3. **Ours, from the ground up.** No third-party code in the tree beyond the exceptions listed in D7. When tempted to vendor something, raise it instead.
4. **Boring before clever.** Pick the design a new reader understands fastest. Cleverness needs a measured reason.
5. **Small slices.** One spec, one reviewable diff, one merge (Q5). If a slice grows past what one person can review in a sitting, split it.
6. **Decisions are written down.** Design choices live in `docs/decisions/`. Changing one is a dated, approved edit (Q13), never a quiet deviation in code.
7. **Main always boots.** Every commit on `main` builds and passes boot tests on every required port (Q11).
8. **Explainable.** Each merged subsystem gets a walkthrough note Graeson can read and explain back (Q14). Code nobody can explain is not done.
9. **Native first, POSIX on top.** Our own API is the real interface (D6). The POSIX layer adapts to it, never the other way round.
10. **Measure, then tune.** Performance work follows a benchmark number (Q10), not a hunch.
