# Identity and goals

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| G1 | What is it called? | Final name now · Working codename until the shell runs · Reuse the bongOS name | yart. | Graeson |
| G2 | How do the goals rank: learning, daily driving, portfolio? | Any order of the three | Daily driving first, then learning, then portfolio. | Graeson |
| G3 | What does "daily drive" mean concretely? | Terminal work only · Plus GUI apps we write · Plus a real web browser | All three: terminal work, our own GUI apps, and a real web browser. The browser is a post-1.0 goal. | Graeson, detailed by Claude |
| G4 | Who should be able to run it? | Only you · Hobbyists who clone and build · Anyone, with an installer | Hobbyists first (one command builds and boots it), anyone eventually, with an installer in phase 6. | Graeson, detailed by Claude |
| G5 | Which license? | MIT · BSD 2-clause · Apache 2.0 · GPLv3 | MIT, matching bongOS. | Claude's pick, approved |
| G6 | Public from the first commit? | Public from day one · Private until phase 3 | Public from day one. The commit history is part of the portfolio. | Claude's pick, approved |
| G7 | What is explicitly out of scope? | Linux binary compatibility · Windows apps · Touch and mobile · Server workloads · Hostile multi-user security | Non-goals: Windows apps, touch and mobile, server workloads, hostile multi-user security. Linux binary compatibility is a wanted stretch goal after 1.0, so the kernel keeps a per-process syscall personality hook from the start. | Graeson, detailed by Claude |
| G8 | How are releases numbered and named? | 0.N per phase · Date based · Semantic versions | 0.N per phase with a codename each, same tradition as bongOS. | Claude's pick, approved |
| G9 | Is there a written list of design principles? | One-page principles file · None | Write it. Example rules: no architecture-specific code outside `arch/`, every feature ships with a test, boring before clever. | Claude's pick, approved |
| G10 | How much time per week, and what happens during exams? | Steady weekly hours · Bursts over breaks | Steady weekly hours. | Graeson |
| G11 | What documentation exists beyond code? | Design note per subsystem · User manual · Devlog | A design note per subsystem in `docs/` plus a devlog. The manual waits for phase 3. | Claude's pick, approved |
