# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Shipyard is an orchestration framework that coordinates multiple Claude Code instances working on the same repository. It uses GitHub (Discussions, Issues, PRs, Labels, Assignees) as the persistent state layer and communication bus. The project is currently in **Phase 0: Specification** -- all files are markdown specifications with no implementation code yet.

## Repository Structure

All files are top-level markdown specifications:

- **SPEC.md** - Complete technical specification (start here for full detail)
- **ARCHITECTURE.md** - System design, data flow diagrams, component interactions
- **ROADMAP.md** - Implementation phases (0-6), currently at Phase 0
- **overview.md** - Skill architecture patterns (polling, claiming, work, completion)
- **commands.md** - CLI specification for the `sy` command
- **labels.md** - GitHub label taxonomy (14 labels across 3 categories + optional type labels)
- **discussions.md** - GitHub Discussions category configuration
- **templates.md** - Issue/PR template specifications
- **merge.md** - Auto-merge logic specification
- **pm.md, coder.md, reviewer.md** - Individual agent persona specifications

## Core Architecture

### Three Stateless Agent Personas

Each agent polls GitHub for work matching specific labels, claims it via assignees, executes, and exits:

| Agent | Polls For | Creates |
|-------|-----------|---------|
| **PM** | Discussions needing planning | Issues with specs |
| **Coder** | Issues with `status:ready` | PRs with code and docs |
| **Reviewer** | PRs with `status:review` | Review comments/approvals/rejections |

Merging is handled by GitHub branch protection rules and auto-merge.

### Label-Based State Machine

Labels encode workflow state. Agents read labels to find work and write labels to signal transitions. Key flow:

```
Discussion (no status) -> PM -> status:planning -> Issues created
-> status:ready -> Coder -> status:in-progress -> PR created
-> status:review -> Reviewer -> status:approved (or status:changes-requested)
-> Auto-merge (branch protection)
```

### Key Design Patterns

- **GitHub as Brain**: All state, communication, and audit trail live in GitHub primitives
- **Assignee-Based Locking**: Agents claim work via assignees (not labels) to avoid TOCTOU race conditions
- **Subagent Pattern**: Agents spawn subagents for analysis to avoid polluting main context (only the summary returns)
- **Git Worktrees**: Agents use `.worktrees/` for isolated file changes
- **Per-Session Token Cap**: Each agent session has a configurable token limit; no cross-session tracking
- **No Self-Approval**: Agents cannot review/approve their own work
- **Agent Identity Prefix**: All communications prefixed with `[PM]`, `[CODER]`, or `[REVIEWER]`
- **Docs in PR**: Documentation updates are included in feature PRs
- **Recovery Paths**: Stale claims (30 min), blocked items (48h), review cycles (3 max) all have defined escalation

### Target Repository Layout (after `sy init`)

```
repo/
├── .shipyard/
│   ├── config.yaml       # Shipyard configuration
│   ├── skills/           # Local skill overrides
│   └── templates/        # Issue/PR templates
├── .worktrees/           # Git worktrees for agent isolation
└── .github/
    ├── ISSUE_TEMPLATE/
    └── PULL_REQUEST_TEMPLATE.md
```

## CLI Specification

The `sy` command (alias for `shipyard`) wraps `gh`, `git`, and `claude` CLIs. Key commands:

- `sy init` - Initialize repo with config, labels, templates
- `sy run <agent>` - Manually trigger an agent (pm, coder, reviewer)
- `sy status` - Show work item state
- `sy queue <agent>` - Show agent's work queue
- `sy merge` - Check and execute merges for approved PRs
- `sy worktrees` - Manage git worktrees

Config lives in `.shipyard/config.yaml`. Environment variables: `SHIPYARD_CONFIG`, `SHIPYARD_DEBUG`, `GITHUB_TOKEN`.

## Contributing

- Branch from `main`
- Markdown specs should use consistent heading levels and include code examples
- Skills should follow patterns in agent specification files (polling -> claiming -> work -> completion)
