# yart

An operating system built from the ground up: kernel, command line and GUI, with no third-party code in the tree. It targets x86_64 and RISC-V 64 from the first commit, AArch64 after that, and eventually a home-built CPU.

**Status: planning complete, no code yet.** Phase 0 specs are next. See [docs/status.md](docs/status.md).

## What it will be

- A monolithic kernel in C with loadable modules, portable across CPU architectures through a small arch contract
- Its own bootloader on UEFI machines and its own machine-mode firmware on RISC-V
- Its own native system API, with a POSIX layer on top so ordinary C programs compile
- Its own filesystem, yartfs
- A shell that speaks POSIX `sh` and passes structured records between native tools
- A software-rendered desktop: compositor, vector graphics, fonts and toolkit, all written here

## How it is built

Every design question was answered before any code: 248 decisions in [docs/decisions/](docs/decisions/). Work then proceeds in small slices, each one going through a fixed pipeline of Claude Code agents (spec, tests, implementation, review, port check, walkthrough) with a human approving the spec and the merge. The pipeline is described in [docs/workflow.md](docs/workflow.md) and defined in [.claude/](.claude/).

## Where to look

| Path | What is there |
| --- | --- |
| [docs/roadmap.md](docs/roadmap.md) | The seven phases and what each one delivers |
| [docs/architecture.md](docs/architecture.md) | Layers, repository layout, structural rules |
| [docs/decisions/](docs/decisions/) | Every settled design decision, by area |
| [docs/design/](docs/design/) | One design note per subsystem |
| [docs/specs/](docs/specs/) | One spec per slice of work |
| [docs/walkthroughs/](docs/walkthroughs/) | Plain-language explanations of what has been built |
| [docs/principles.md](docs/principles.md) | The rules that break ties |
| [docs/open-questions.md](docs/open-questions.md) | What is still undecided |
| [CLAUDE.md](CLAUDE.md) | Operating instructions for Claude Code sessions |

## License

MIT. See [LICENSE](LICENSE).
