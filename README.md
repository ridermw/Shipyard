# Shipyard

A lightweight orchestration framework for Claude Code that coordinates multiple agent personas working on the same repository. GitHub serves as the persistent state layer and communication bus, while stateless Claude Code skills poll for work and execute tasks.

## Philosophy

Shipyard embraces simplicity over complexity. Rather than building elaborate orchestration infrastructure, it leverages tools you already trust:

**GitHub as the Brain**: Discussions for planning, Issues for tracking, PRs for review, Labels for workflow state. All communication happens in the open, creating a natural audit trail.

**Claude Code as the Hands**: Stateless skill instances poll GitHub for work matching their persona. Each agent (PM, Coder, Reviewer, Owner, Librarian) operates independently with clear responsibilities.

**Worktrees for Isolation**: When agents need to make file changes asynchronously, Git worktrees keep their work separate until merged.

## Agent Personas

| Agent | Polls For | Creates |
|-------|-----------|---------|
| **PM** | Discussions needing planning | Feature requests, specs |
| **Coder** | High priority feature requests | Branches, PRs |
| **Reviewer** | PRs ready for review | Review comments, approvals |
| **Owner** | Approved PRs | Merge decisions |
| **Librarian** | Merged PRs | Doc updates, changelog entries |

## Quick Start

```bash
# Clone and install (future)
npx shipyard init
# or
sy init

# Start an agent manually (POC approach)
claude --skill shipyard-coder
```

## Project Status

This project is in the planning and specification phase. See [docs/SPEC.md](docs/SPEC.md) for the full specification and [docs/ROADMAP.md](docs/ROADMAP.md) for implementation phases.

## Documentation

| Document | Description |
|----------|-------------|
| [SPEC.md](docs/SPEC.md) | Complete technical specification |
| [ARCHITECTURE.md](docs/ARCHITECTURE.md) | System design and data flow |
| [ROADMAP.md](docs/ROADMAP.md) | Implementation phases |
| [skills/](docs/skills/) | Agent skill specifications |
| [github/](docs/github/) | GitHub configuration (labels, discussions, templates) |
| [cli/](docs/cli/) | Command line interface specification |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License. See [LICENSE](LICENSE) for details.
