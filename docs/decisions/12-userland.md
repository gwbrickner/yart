# Command line and userland

Settled decisions for this area. Cite them by ID in specs, design notes and commit messages. To change one, follow the process in [README.md](README.md).

| ID | Question | Options considered | Decision | Decided by |
| --- | --- | --- | --- | --- |
| U1 | What language does the shell speak? | POSIX `sh` · Our own language · POSIX core with our own interactive extras | POSIX `sh` grammar and semantics for scripts, plus our own interactive extras and structured pipelines (U2). | Claude, following U2 |
| U2 | Do pipelines carry text or structured data? | Plain text · Structured records | Structured records. Native tools exchange typed records through pipes; plain text is the fallback whenever either side is an ordinary POSIX program. | Graeson, detailed by Claude |
| U3 | Which interactive features? | Line editing · History · Tab completion · Syntax highlighting · Suggestions from history | The first three in phase 3. Highlighting and suggestions are stretch goals. | Claude's pick, approved |
| U4 | Is there job control? | Yes · No | Yes: background jobs, suspend, resume. | Claude's pick, approved |
| U5 | How complete is scripting? | Basics only · All of POSIX `sh` | All of POSIX `sh`, checked against a written test list. | Claude's pick, approved |
| U6 | What is the terminal model? | Classic TTY with pseudo-terminals · Something new | Classic TTY with pseudo-terminals. Ported programs assume it. | Claude's pick, approved |
| U7 | Which escape codes does the terminal understand? | Basic VT100 · An xterm-compatible subset with full color | An xterm-compatible subset with full color. One implementation serves the kernel console and the GUI terminal. | Claude's pick, approved |
| U8 | Which utilities ship in phase 3? | A minimal dozen · A full working set | About thirty: file tools, text tools (`grep`, `sort`, `head`, `tail`, `wc`), `find`, `ps`, `kill`, `mount`, `df`, `date`, `less`. Then `sed`, `awk`, `diff` and `tar` as a second batch. | Claude's pick, approved |
| U9 | One binary or many? | One multi-call binary · Separate small programs | Separate programs sharing one utility library. | Claude's pick, approved |
| U10 | What option style do tools use? | POSIX flags · Our own style | POSIX flags on POSIX tools, since scripts depend on them. Our own tools share one long-option style. | Claude's pick, approved |
| U11 | What text editor ships? | Modal, vi-style · Modeless, nano-style · Both | Modeless, nano-style. | Graeson |
| U12 | What starts and supervises services? | Init plus a startup script · A declarative service manager | Init plus a script in phase 3. A robust service manager in phase 5: dependencies, restarts with backoff, readiness, per-service logs, on-demand start. | Graeson, detailed by Claude |
| U13 | What format is system configuration in? | Key=value with sections · TOML · JSON | Key=value with sections, one parser used everywhere. | Claude's pick, approved |
| U14 | How do users get help? | `--help` only · Manual pages | Both. `--help` is generated from each tool's option table, and manual pages use a simple markup. | Claude's pick, approved |
| U15 | Is there a package manager? | None · Simple archives · Full dependency solver | None through phase 5. A full package manager with dependency resolution in phase 6. | Graeson, detailed by Claude |
| U16 | How are locale and time zones handled? | UTF-8 only · Full locale system | UTF-8 only. The time zone database is imported as data in phase 6. | Claude's pick, approved |
| U17 | Where do system logs go? | Kernel buffer only · A log service writing to `/var/log` | A log service writing to `/var/log`, with a viewer tool. | Claude's pick, approved |
| U18 | Which system tools exist beyond the basics? | Process monitor · Hex dump · Syscall tracer | All three by the end of phase 3. | Claude's pick, approved |
