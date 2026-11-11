# Testing, quality and workflow

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| Q1 | Which kinds of tests exist? | Host-side unit tests · In-kernel self tests · Userland test programs · Scripted boot tests in QEMU | All four. Anything that is pure logic (allocators, parsers, filesystem code) also compiles and runs on the host. | Claude's pick, approved |
| Q2 | What does CI run on every push? | Build only · Build and boot every required port | Build and boot every required port, running all four test kinds on each. | Claude's pick, approved |
| Q3 | What is the merge rule? | Tests encouraged · Tests required | Required. Every feature adds a test and every bug fix adds a regression test. | Claude's pick, approved |
| Q4 | What does a milestone spec contain? | A paragraph · Goal, interfaces, behaviors, test list, out of scope | The full five-part spec, one file per milestone in `docs/specs/`, approved by you before a build session starts. | Claude's pick, approved |
| Q5 | How big is one build session's task? | A whole phase · One slice with a reviewable diff | One slice with a reviewable diff. Never a whole phase. | Claude's pick, approved |
| Q6 | Who reviews generated code? | You · A fresh review session · Both | Both. A fresh session checks the diff against the spec, then you read it. | Claude's pick, approved |
| Q7 | Is there a full bug sweep after each phase? | Yes · Only when something breaks | Yes, across every subsystem, before the next spec is written. | Claude's pick, approved |
| Q8 | What gets fuzzed? | Nothing · Parsers · Syscalls | Parsers on the host from the day each exists (ELF, filesystem images, fonts, PNG). Syscalls from phase 3. | Claude's pick, approved |
| Q9 | How is the GUI tested? | By eye · Headless toolkit tests and frame comparison | Headless toolkit tests plus frame comparison through the screenshot call (W29). | Claude's pick, approved |
| Q10 | Which performance numbers are tracked? | None · Boot time, syscall and context-switch cost, frame time | Those four, recorded at each release. | Claude's pick, approved |
| Q11 | How does the branch model work? | Commit to main · Main always boots, work lands by pull request | Main always boots, and work lands by pull request. | Claude's pick, approved |
| Q12 | When is a phase done? | The demo works · A written checklist | A checklist: the phase's milestone works, tests pass on every required port, design notes are current, sweep is complete. | Claude's pick, approved |
| Q13 | How do decisions change? | Edit silently · Dated note with the reason | Dated note with the reason, in this doc. | Claude's pick, approved |
| Q14 | How do you keep understanding what gets built? | Trust the tests · A walkthrough note per subsystem | A walkthrough note per merged subsystem. You should be able to explain it before the next one starts. | Claude's pick, approved |
| Q15 | What is the budget for cloud sessions and CI? | Fixed monthly amount · Per phase · Unbounded | Open. Graeson sets the number; small slices (Q5) are also the cheap ones. | Open |
