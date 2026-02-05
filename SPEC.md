# Shipyard Specification

Version: 0.2.0 (Planning Phase)

## Overview

Shipyard is a lightweight orchestration framework that coordinates multiple Claude Code instances working on the same repository. Each instance runs a specialized "skill" representing an agent persona (PM, Coder, Reviewer). GitHub serves as the persistent state layer, communication bus, and audit trail. Merging is handled by branch protection rules and auto-merge logic.

## Design Principles

**Stateless Agents**: Each Claude Code instance polls GitHub for work, executes a task, updates GitHub, and terminates. No persistent memory beyond what GitHub stores.

**One-Shot Execution**: Each agent invocation handles one task (or discovers no work). External orchestration (shell scripts, cron, GitHub Actions) provides the loop. Agents do not loop internally. See [orchestration.md](orchestration.md).

**GitHub as Truth**: All communication, planning, and state transitions happen through GitHub primitives (Discussions, Issues, PRs, Labels, Comments, Assignees). This creates a natural audit trail and allows human oversight at any point.

**Token Prudence**: Agents work in isolated contexts. Subagents answer questions without polluting the main context. Each agent invocation has a budget cap enforced via `--max-budget-usd`.

**Skills + Hooks**: Skills provide agent persona and context. Hooks enforce hard constraints (blocking forbidden operations). See [hooks.md](hooks.md).

**Progressive Automation**: Start with manual orchestration (running agents in separate terminals), then progressively automate with cron/polling as patterns stabilize.

## System Components

### 1. GitHub Infrastructure

GitHub provides the "database" and "message bus" for Shipyard:

| Component | Purpose | Agents |
|-----------|---------|--------|
| **Discussions** | Planning, design, retrospectives | PM creates, all can comment |
| **Issues** | Feature requests, bugs, tasks | PM creates, Coder consumes |
| **Pull Requests** | Code changes | Coder creates, Reviewer consumes |
| **Labels** | Workflow state and priority | All agents read/write |
| **Assignees** | Work item locking | All agents use for claiming |
| **Comments** | Agent communication | All agents |

See [labels.md](labels.md) and [discussions.md](discussions.md) for detailed configuration.

### 2. Agent Skills

Each agent is a Claude Code skill invoked via `claude -p`:

1. Polls GitHub for items matching specific labels/states
2. Claims work by assigning itself and verifying sole assignee
3. Performs its specialized task
4. Updates GitHub with results (comments, new items, label changes)
5. Releases the claim
6. Terminates (via `--max-turns` limit)

External orchestration handles the polling loop. See [orchestration.md](orchestration.md).

| Agent | Skill File | Responsibility |
|-------|------------|----------------|
| PM | `shipyard-pm` | Plans features, creates specs, manages discussions |
| Coder | `shipyard-coder` | Implements features, creates PRs, updates docs, resolves merge conflicts |
| Reviewer | `shipyard-reviewer` | Reviews code, verifies intent, checks scope, approves or rejects |

See [overview.md](overview.md) for detailed skill specifications.

### 3. Merge Logic

Merging is handled by GitHub branch protection rules and an optional auto-merge script, since merging is a deterministic action once review and CI pass. See [merge.md](merge.md) for details.

### 4. Worktrees

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

### 5. Agent Identity

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
2. PM skill polls for Discussions in "Planning" with no `status:*` label
3. PM claims the Discussion (assigns self, adds `status:planning`)
4. PM analyzes the discussion, creates a detailed spec
5. PM creates Issue(s) tagged `type:feature`, `priority:high`, `status:ready`
6. PM closes/answers the original Discussion

### Phase 2: Implementation (Coder)

1. Coder skill polls for Issues with `status:ready`
2. Coder claims Issue (assigns self, verifies sole assignee, adds `status:in-progress`, `agent:coder`)
3. Coder creates worktree, implements feature
4. Coder updates relevant documentation as part of the same PR
5. Coder creates PR linked to Issue, adds `status:review`

### Phase 3: Review (Reviewer)

1. Reviewer skill polls for PRs with `status:review`
2. Reviewer claims PR (assigns self, verifies, adds `agent:reviewer`)
3. Reviewer reads linked Issue and Discussion for intent verification
4. Reviewer verifies CI has passed
5. Reviewer checks scope (PR only addresses linked Issue)
6. Reviewer checks that docs are updated for user-facing changes
7. Reviewer either requests changes (`status:changes-requested`) or approves (`status:approved`)
8. If fundamentally misaligned: Reviewer closes PR, returns Issue to `status:ready`
9. After 3 review cycles without resolution: `status:needs-human`

### Phase 4: Merge

1. PR with `status:approved` is eligible for merge
2. Branch protection rules verify: review approved, CI passed, no conflicts
3. Auto-merge (GitHub native or `sy merge` script) squash-merges the PR
4. Linked Issue(s) are auto-closed via `Closes #N` in commit message
5. If merge fails (conflicts): PR gets `status:blocked`, Coder picks up for rebase

## Budget Management

Budget is enforced externally via Claude Code CLI flags, not internally by agents.

### Per-Invocation Limits

```bash
claude -p "..." \
  --max-turns 20 \           # Force termination after N API turns
  --max-budget-usd 5.00      # Hard budget cap per invocation
```

| Agent | Recommended `--max-turns` | Recommended `--max-budget-usd` |
|-------|---------------------------|-------------------------------|
| PM | 10 | 2.00 |
| Coder | 20 | 5.00 |
| Reviewer | 15 | 3.00 |

### Why External Enforcement

Claude Code does not have a native "exit" signal. The `--max-turns` flag forces termination after N API round-trips, providing predictable resource usage.

If budget or turns are exhausted mid-task:

1. Agent terminates immediately
2. Claim remains on item (assignee still set)
3. After 30 minutes with no status update, stale claim detection clears the assignee
4. Item returns to ready state for next invocation

### Cross-Session Tracking

There is no cross-session budget tracking. Each agent invocation is independent. Cost management across sessions is handled externally (Anthropic API dashboard, billing alerts, orchestration script monitoring).

See [orchestration.md](orchestration.md) for detailed configuration.

## Security Considerations

**No Self-Approval**: Enforced via hooks that block `gh pr review --approve` for non-Reviewer agents. Skills also instruct agents not to approve their own work, providing defense in depth. See [hooks.md](hooks.md).

**Human Override**: Humans can intervene at any point by adding/removing labels, closing items, or leaving comments that agents respect.

**Audit Trail**: All actions are recorded in GitHub, providing complete traceability.

**Budget Limits**: `--max-budget-usd` and `--max-turns` flags prevent runaway costs.

**Hooks for Enforcement**: Hard constraints (no force push to main, no destructive git commands) are enforced via Claude Code hooks, not just skill instructions. See [hooks.md](hooks.md).

**Prompt Injection Defense**: Agent skill instructions include defensive prompts to guard against injection via Issue/PR content. Agents treat all GitHub content (Issue bodies, PR descriptions, comments) as untrusted input. Skill prompts instruct agents to ignore any instructions embedded in work item content that attempt to override agent behavior. Hooks provide an additional layer by blocking specific dangerous operations regardless of instructions.

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

See [commands.md](commands.md) for full specification.

## Related Documents

- [orchestration.md](orchestration.md) — How agents are invoked and coordinated
- [hooks.md](hooks.md) — Constraint enforcement via Claude Code hooks
- [labels.md](labels.md) — Label taxonomy and claiming mechanism
- [overview.md](overview.md) — Skill architecture patterns

## Future Considerations

These are out of scope for POC but inform design decisions:

1. **Distributed Nodes**: Push work to other machines
2. **Heartbeat/Health**: Monitor agent health, restart failed tasks
3. **Scheduled Runs**: GitHub Actions or cron for automated polling
4. **Custom Agents**: User-defined agent personas beyond the core three
5. **Cross Repo**: Agents working across multiple repositories
6. **Additional Agents**: User-defined agent personas for dedicated merge oversight, documentation management, or other specialized workflows
