# Shipyard

A lightweight orchestration framework for Claude Code that coordinates multiple agent personas working on the same repository. GitHub serves as the persistent state layer and communication bus, while stateless Claude Code skills poll for work and execute tasks.

## Philosophy

Shipyard embraces simplicity over complexity. Rather than building elaborate orchestration infrastructure, it leverages tools you already trust:

**GitHub as the Brain**: Discussions for planning, Issues for tracking, PRs for review, Labels for workflow state. All communication happens in the open, creating a natural audit trail.

**Claude Code as the Hands**: Stateless skill instances poll GitHub for work matching their persona. Each agent (PM, Coder, Reviewer) operates independently with clear responsibilities. Merging is handled by branch protection rules.

**Worktrees for Isolation**: When agents need to make file changes asynchronously, Git worktrees keep their work separate until merged.

## Agent Personas

| Agent | Polls For | Creates |
|-------|-----------|---------|
| **PM** | Discussions needing planning | Issues with specs |
| **Coder** | Issues ready for implementation | PRs with code and docs |
| **Reviewer** | PRs ready for review | Review comments, approvals, rejections |

Coder also handles documentation updates (included in feature PRs) and merge conflict resolution. Reviewer also handles intent verification and scope checking. Merging of approved PRs is handled by GitHub branch protection rules and auto-merge.

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

This project is in the planning and specification phase. See [SPEC.md](SPEC.md) for the full specification and [ROADMAP.md](ROADMAP.md) for implementation phases.

## Documentation

| Document | Description |
|----------|-------------|
| [SPEC.md](SPEC.md) | Complete technical specification |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design and data flow |
| [ROADMAP.md](ROADMAP.md) | Implementation phases |
| [overview.md](overview.md) | Skill architecture patterns |
| [pm.md](pm.md) | PM agent specification |
| [coder.md](coder.md) | Coder agent specification |
| [reviewer.md](reviewer.md) | Reviewer agent specification |
| [merge.md](merge.md) | Merge logic specification |
| [labels.md](labels.md) | GitHub label taxonomy |
| [discussions.md](discussions.md) | GitHub Discussions configuration |
| [templates.md](templates.md) | Issue/PR template specifications |
| [commands.md](commands.md) | CLI specification |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License. See [LICENSE](LICENSE) for details.
