# Shipyard Specification

Version: 0.1.0 (Planning Phase)

## Overview

Shipyard is a lightweight orchestration framework that coordinates multiple Claude Code instances working on the same repository. Each instance runs a specialized "skill" representing an agent persona (PM, Coder, Reviewer, Owner, Librarian). GitHub serves as the persistent state layer, communication bus, and audit trail.

## Design Principles

**Stateless Agents**: Each Claude Code instance polls GitHub for work, executes a task, updates GitHub, and terminates. No persistent memory beyond what GitHub stores.

**GitHub as Truth**: All communication, planning, and state transitions happen through GitHub primitives (Discussions, Issues, PRs, Labels, Comments). This creates a natural audit trail and allows human oversight at any point.

**Token Prudence**: Agents work in isolated contexts. Subagents answer questions without polluting the main context. Each agent stops if overall usage approaches 80% of budget.

**Progressive Automation**: Start with manual orchestration (running skills in separate terminals), then progressively automate with cron/polling as patterns stabilize.

## System Components

### 1. GitHub Infrastructure

GitHub provides the "database" and "message bus" for Shipyard:

| Component | Purpose | Agents |
|-----------|---------|--------|
| **Discussions** | Planning, design, retrospectives | PM creates, all can comment |
| **Issues** | Feature requests, bugs, tasks | PM creates, Coder consumes |
| **Pull Requests** | Code changes | Coder creates, Reviewer/Owner consume |
| **Labels** | Workflow state and priority | All agents read/write |
| **Comments** | Agent communication | All agents |
| **Wiki/Docs** | Persistent documentation | Librarian maintains |

See [github/labels.md](github/labels.md) and [github/discussions.md](github/discussions.md) for detailed configuration.

### 2. Agent Skills

Each agent is a Claude Code skill that:

1. Polls GitHub for items matching specific labels/states
2. Claims work by adding an "in progress" label
3. Performs its specialized task
4. Updates GitHub with results (comments, new items, label changes)
5. Exits cleanly

| Agent | Skill File | Responsibility |
|-------|------------|----------------|
| PM | `shipyard-pm` | Plans features, creates specs, manages discussions |
| Coder | `shipyard-coder` | Implements features, creates PRs |
| Reviewer | `shipyard-reviewer` | Reviews code, requests changes, approves |
| Owner | `shipyard-owner` | Makes merge decisions, enforces quality |
| Librarian | `shipyard-librarian` | Updates docs, maintains changelog |

See [skills/overview.md](skills/overview.md) for detailed skill specifications.

### 3. Worktrees

Git worktrees provide isolation when agents make file changes:

```
repo/
├── .git/                    # Shared git database
├── (main working tree)      # Human development
└── .worktrees/
    ├── coder-feature-123/   # Coder working on feature
    └── coder-feature-456/   # Coder working on another feature
```

**Lifecycle**:
1. Coder creates worktree when starting a feature
2. Worktree persists until PR is merged and closed
3. Cleanup happens after merge

### 4. Agent Identity

All agents operate under a single GitHub username but identify themselves in communications:

**Commit Messages**: `[CODER] Implement user authentication`

**PR Descriptions**: `[CODER] This PR implements...`

**Comments**: `[REVIEWER] I have concerns about...`

**Labels**: `agent:coder`, `agent:reviewer`, etc.

This allows tracing which agent performed which action in the audit trail.

## Workflow Example

A complete feature lifecycle demonstrates how agents coordinate:

### Phase 1: Planning (PM)

1. Human or PM creates a Discussion in "Planning" category
2. PM skill polls for Discussions lacking `status:planned` label
3. PM analyzes the discussion, creates a detailed spec
4. PM creates Issue(s) tagged `type:feature`, `priority:high`, `status:ready-for-coder`
5. PM adds `status:planned` to original Discussion

### Phase 2: Implementation (Coder)

1. Coder skill polls for Issues with `status:ready-for-coder`
2. Coder claims issue by adding `status:in-progress`, `agent:coder`
3. Coder creates worktree, implements feature
4. Coder creates PR linked to Issue, adds `status:ready-for-review`
5. Coder removes `status:in-progress` from Issue

### Phase 3: Review (Reviewer)

1. Reviewer skill polls for PRs with `status:ready-for-review`
2. Reviewer claims PR by adding `agent:reviewer`
3. Reviewer analyzes code, adds comments
4. Reviewer either requests changes (`status:changes-requested`) or approves (`status:approved`)

### Phase 4: Merge Decision (Owner)

1. Owner skill polls for PRs with `status:approved`
2. Owner reviews original Discussion for intent adherence
3. Owner either merges or rejects with explanation
4. Owner closes linked Issue(s)

### Phase 5: Documentation (Librarian)

1. Librarian skill polls for recently merged PRs
2. Librarian reviews changes and original Discussion
3. Librarian updates relevant docs, README, changelog
4. Librarian creates PR for doc changes (follows same review flow)

## Token Budget Management

Each agent skill includes budget awareness:

```
On startup:
  1. Check current usage against 80% threshold
  2. If over threshold, log warning and exit gracefully
  3. If under threshold, proceed with work

During work:
  1. Periodically check usage
  2. If approaching threshold, wrap up current task and exit
  3. Leave breadcrumbs in GitHub for next invocation
```

## Security Considerations

**No Self Approval**: Skills enforce that an agent cannot approve its own work. The Coder skill will not review PRs it created.

**Human Override**: Humans can intervene at any point by adding/removing labels, closing items, or leaving comments that agents respect.

**Audit Trail**: All actions are recorded in GitHub, providing complete traceability.

**Token Limits**: Budget awareness prevents runaway costs.

## CLI Interface

The `shipyard` (alias: `sy`) command provides:

```bash
# Initialize a repo for Shipyard
sy init

# Check status of all agents/work
sy status

# Manually trigger an agent
sy run pm
sy run coder
sy run reviewer

# View pending work
sy queue

# Update skills from upstream
sy update
```

See [cli/commands.md](cli/commands.md) for full specification.

## Future Considerations

These are out of scope for POC but inform design decisions:

1. **Distributed Nodes**: Push work to other machines
2. **Heartbeat/Health**: Monitor agent health, restart failed tasks
3. **Scheduled Runs**: GitHub Actions or cron for automated polling
4. **Custom Agents**: User defined agent personas
5. **Cross Repo**: Agents working across multiple repositories
