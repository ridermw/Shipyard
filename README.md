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

This walkthrough takes you from a new Discussion to merged code.

### Prerequisites

```bash
# GitHub CLI authenticated
gh auth status

# Claude Code installed
claude --version
```

### 1. Initialize Repository

```bash
cd your-repo
sy init

# Creates:
# - .shipyard/config.yaml
# - .shipyard/hooks/
# - .claude/settings.json
# - GitHub labels (14 labels)
```

### 2. Create a Feature Request

Create a GitHub Discussion in the "Planning" category:

> **Title**: Add user logout button
>
> **Body**: Users need a way to log out from the application. The logout button should be in the header, clear the session, and redirect to the login page.

### 3. Run the PM Agent

```bash
sy run pm

# PM will:
# 1. Find the Discussion (no status label)
# 2. Add status:planning label
# 3. Analyze the request
# 4. Create Issue #1 with acceptance criteria
# 5. Close the Discussion
```

**Result**: Issue #1 created with `status:ready`, `priority:medium`, `type:feature`

### 4. Run the Coder Agent

```bash
sy run coder

# Coder will:
# 1. Find Issue #1 with status:ready
# 2. Claim it (assign self, add status:in-progress)
# 3. Create worktree .worktrees/feature-1
# 4. Implement the logout button
# 5. Create PR #2 with status:review
```

**Result**: PR #2 created, linked to Issue #1

### 5. Run the Reviewer Agent

```bash
sy run reviewer

# Reviewer will:
# 1. Find PR #2 with status:review
# 2. Claim it (assign self)
# 3. Read Issue #1 and original Discussion for intent
# 4. Review code, check CI status
# 5. Approve with status:approved (or request changes)
```

**Result**: PR #2 approved and ready to merge

### 6. Merge

```bash
sy merge

# Or let GitHub auto-merge handle it
# Branch protection verifies: approved, CI green, no conflicts
```

**Result**: PR #2 merged, Issue #1 auto-closed

### Complete Workflow Script

For automated runs, chain the agents:

```bash
#!/bin/bash
# shipyard-run.sh

set -e

# Clear any stale claims first
sy status --stale --fix

# Run each agent
sy run pm
sy run coder
sy run reviewer
sy merge

echo "Workflow complete"
```

Or run via cron every 5 minutes:

```cron
*/5 * * * * /path/to/shipyard-run.sh >> /var/log/shipyard.log 2>&1
```

### What's Actually Happening

Each `sy run <agent>` invokes Claude Code:

```bash
claude -p "$(cat .shipyard/skills/coder/SKILL.md) ..." \
  --max-turns 20 \
  --max-budget-usd 5.00 \
  --allowedTools "Bash(gh *),Bash(git *),Read,Edit,Write" \
  --no-session-persistence
```

Agents are **one-shot**: each invocation handles one task (or finds no work), then terminates. The orchestration script provides the loop. See [orchestration.md](orchestration.md) for details.

## Project Status

This project is in the planning and specification phase. See [SPEC.md](SPEC.md) for the full specification and [ROADMAP.md](ROADMAP.md) for implementation phases.

## Documentation

### Core Specs

| Document | Description |
|----------|-------------|
| [SPEC.md](SPEC.md) | Complete technical specification |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System design and data flow |
| [ROADMAP.md](ROADMAP.md) | Implementation phases |

### Execution Model

| Document | Description |
|----------|-------------|
| [orchestration.md](orchestration.md) | External orchestration, Claude Code invocation, budget enforcement |
| [hooks.md](hooks.md) | Constraint enforcement via Claude Code hooks |
| [overview.md](overview.md) | Skill architecture patterns |

### Agent Specifications

| Document | Description |
|----------|-------------|
| [pm.md](pm.md) | PM agent specification |
| [coder.md](coder.md) | Coder agent specification |
| [reviewer.md](reviewer.md) | Reviewer agent specification |

### GitHub Configuration

| Document | Description |
|----------|-------------|
| [labels.md](labels.md) | GitHub label taxonomy and claiming mechanism |
| [discussions.md](discussions.md) | GitHub Discussions configuration |
| [templates.md](templates.md) | Issue/PR template specifications |
| [merge.md](merge.md) | Merge logic specification |

### CLI

| Document | Description |
|----------|-------------|
| [commands.md](commands.md) | CLI specification for `sy` command |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License. See [LICENSE](LICENSE) for details.
