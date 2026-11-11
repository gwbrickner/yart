# Security and users

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| X1 | What are we defending against? | Nothing · Buggy programs · Network attackers · Hostile local users | Buggy programs first, network-facing code second. Hostile local users stay a non-goal (G7). | Claude's pick, approved |
| X2 | Is it multi-user? | Single user · Unix users and groups | Unix users and groups from phase 2. Cheap now, painful to retrofit. | Claude's pick, approved |
| X3 | What is the privilege model? | All-powerful root · Fine-grained capabilities | Root, with a capability split considered after 1.0. | Claude's pick, approved |
| X4 | How does a user gain privilege? | Setuid programs · An elevation service | An elevation service. No setuid binaries at all. | Claude's pick, approved |
| X5 | How does login work? | Auto-login · Passwords | Auto-login through phase 4, hashed passwords with the GUI login screen in phase 5. | Claude's pick, approved |
| X6 | Which memory protections are on? | No writable-and-executable pages · Randomized user layout · Stack protectors | All three, from phase 2. | Claude's pick, approved |
| X7 | Can programs restrict themselves? | No · A pledge-style call that drops syscalls and paths · Full containers | A pledge-style call in phase 5. Small to build, and every GUI app can use it. | Claude's pick, approved |
| X8 | Where does cryptography come from? | Our own · A ported, vetted library | Hashes can be ours, checked against published test vectors. Ciphers and TLS come from a ported library in phase 6. | Claude's pick, approved |
| X9 | Is there disk encryption or secure boot? | Yes · Not before 1.0 | Not before 1.0. | Claude's pick, approved |
| X10 | Are speculative-execution attacks mitigated? | Yes · Documented non-goal | Documented non-goal before 1.0. | Claude's pick, approved |
