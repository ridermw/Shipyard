# Shipyard Architecture

This document explains the system design, data flow, and component interactions in Shipyard.

## High Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         GitHub                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ Discussions │  │   Issues    │  │    PRs      │             │
│  │  (Planning) │  │  (Tracking) │  │  (Changes)  │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          │                                      │
│              Labels + Assignees + Comments                      │
│              (State Machine + Locking)                          │
└──────────────────────────┬──────────────────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │      GitHub API         │
              │    (gh CLI / REST)      │
              └────────────┬────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Claude Code  │  │  Claude Code  │  │  Claude Code  │
│   (PM Skill)  │  │(Coder Skill)  │  │(Reviewer...)  │
└───────────────┘  └───────┬───────┘  └───────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
             ┌──────────┐  ┌──────────────┐
             │Git Work- │  │ Auto-Merge   │
             │  tree    │  │   Logic      │
             │(Isolated)│  │(Branch Prot.)│
             └──────────┘  └──────────────┘
```

## Data Flow

### State Transitions via Labels and Assignees

Labels encode the workflow state machine. Assignees provide the locking mechanism. Agents read labels to find work, claim via assignees, and write labels to signal state changes.

```
Discussion Created (no status label)
       │
       ▼
[status:planning] ──PM──▶ Discussion closed/answered
                                 │
                                 ▼
                         Issue(s) Created
                                 │
                                 ▼
                          [status:ready]
                                 │
                          ──Coder──
                                 │
                                 ▼
                        [status:in-progress]
                                 │
                                 ▼
                            PR Created
                                 │
                                 ▼
                          [status:review]
                                 │
                          ──Reviewer──
                                 │
                  ┌──────────────┴──────────────┐
                  ▼                              ▼
      [status:changes-requested]         [status:approved]
                  │                              │
           ──Coder──                      Auto-Merge
                  │                         (branch
                  ▼                       protection)
           [status:review]                       │
                                                 ▼
                                              Merged
                                        (GitHub native)
```

### Recovery Paths

```
                   ┌──48h timeout──▶ [status:needs-human]
[status:blocked] ──┤
                   ├──Coder rebases──▶ [status:review]    (merge conflict)
                   └──Coder fixes──▶ [status:review]      (CI failure)

[status:changes-requested] ──3 cycles──▶ [status:needs-human]

PR fundamentally misaligned ──▶ PR closed ──▶ Issue → [status:ready]
                                               (or [status:needs-human])

Agent crash ──30min stale──▶ Claim cleared ──▶ Previous ready state
```

### Agent Polling Patterns

Each agent polls for specific label and assignee combinations:

| Agent | Polls For | Claims With |
|-------|-----------|-------------|
| PM | Discussions in "Planning" with no `status:*` label | Assign self + `status:planning` |
| Coder | Issues with `status:ready` (new work) | Assign self + `status:in-progress`, `agent:coder` |
| Coder | Own PRs with `status:blocked` (merge conflicts, CI) | Already assigned |
| Coder | Own PRs with `status:changes-requested` | Already assigned |
| Reviewer | PRs with `status:review` AND NOT `author:self` | Assign self + `agent:reviewer` |
| Auto-Merge | PRs with `status:approved` + CI pass + no conflicts | Branch protection rules (no agent) |

### Claiming via Assignees

GitHub label operations are **not** atomic. Two agents can read the same item as unclaimed and both attempt to add a label, creating a race condition.

Shipyard uses the assignee field as an optimistic lock:

```
Agent A                          Agent B
   │                                │
   ├─ Assign self to Issue #42      │
   │                                ├─ Assign self to Issue #42
   │                                │
   ├─ Re-read Issue #42             │
   │  → sees: [Agent A, Agent B]    ├─ Re-read Issue #42
   │  → conflict! Back off          │  → sees: [Agent A, Agent B]
   │  → remove self                 │  → conflict! Back off
   │                                │  → remove self
   │                                │
   ├─ Retry (or pick next item)     │
   ...                              ...
```

In the worst case, both agents back off. This is safe (no duplicate work) and rare in practice since agents poll on different schedules.

### Subagent Pattern

When an agent needs to answer a question without polluting its main context, it spawns a subagent:

```
┌─────────────────────────────────────┐
│         Main Agent Context          │
│                                     │
│  "I need to understand the auth     │
│   module before making changes"     │
│                                     │
│         ┌───────────────┐           │
│         │   Subagent    │           │
│         │  (New Context)│           │
│         │               │           │
│         │ Reads files,  │           │
│         │ analyzes code │           │
│         │               │           │
│         │ Returns:      │           │
│         │ "Auth uses    │           │
│         │  JWT tokens"  │           │
│         └───────┬───────┘           │
│                 │                   │
│         Summary returned            │
│         (tokens saved)              │
│                                     │
└─────────────────────────────────────┘
```

The main agent receives only the answer, not the full reasoning chain.

## Merge Logic

Merging is handled by GitHub branch protection rules and an optional auto-merge script.

### Branch Protection Requirements

- Require at least 1 approving review
- Require CI status checks to pass
- Require linear history (squash merge)
- No force pushes to main

### Auto-Merge Behavior

When a PR reaches `status:approved`:

1. Branch protection ensures CI has passed and review is approved
2. GitHub auto-merge (or `sy merge` script) squash-merges the PR
3. Linked Issues are auto-closed via commit message (`Closes #N`)
4. If merge fails (conflicts), PR is labeled `status:blocked` with a `[SYSTEM]` comment explaining the reason. Coder picks it up for rebase.

See [merge.md](merge.md) for full merge logic specification.

## CI Integration

CI must pass before a PR can be approved or merged:

- **CI pending**: Reviewer may begin review but cannot approve until CI passes
- **CI failure**: PR is labeled `status:blocked` with a comment. Coder investigates.
- **CI pass + review approved**: PR is eligible for merge

CI status is checked via GitHub status checks or check runs, not by Shipyard labels. The `status:blocked` label is only added when an agent or the merge script detects a failure that requires action.

## Stale Claim Detection

Stale claims are detected and cleared to prevent items from getting permanently stuck:

| Condition | Timeout | Action |
|-----------|---------|--------|
| Item has assignee, no status change | 30 min | Clear assignee, remove `agent:*` label, return to ready state |
| `status:blocked` with no update | 48 hours | Escalate to `status:needs-human` |

The `sy status --stale` command reports stale items. A cron job or GitHub Action can automate stale claim cleanup.

## File System Layout

### Target Repository (After `sy init`)

```
your-repo/
├── .shipyard/
│   ├── config.yaml          # Shipyard configuration
│   ├── skills/              # Local skill overrides (optional)
│   └── templates/           # Issue/PR templates
├── .worktrees/              # Git worktrees for agents
│   └── (created on demand)
└── (your existing code)
```

### Shipyard Framework Repository

```
Shipyard/
├── docs/                    # Specifications
├── skills/                  # Core skill definitions
│   ├── shipyard-pm/
│   ├── shipyard-coder/
│   └── shipyard-reviewer/
├── scripts/                 # CLI implementation
│   ├── sy.sh               # Main entry point
│   ├── init.sh             # Initialize a repo
│   └── (other commands)
└── templates/               # Default templates
```

## Communication Patterns

### Synchronous (Within Session)

Agents use subagents for immediate answers:

```
Coder needs to understand test patterns
  └─▶ Subagent reads test files
  └─▶ Returns summary
  └─▶ Coder continues with context
```

### Asynchronous (Across Sessions)

Agents communicate via GitHub:

```
PM finishes planning
  └─▶ Creates Issue with labels
  └─▶ PM session ends

(Later)

Coder polls GitHub
  └─▶ Finds new Issue
  └─▶ Reads Discussion for context
  └─▶ Begins implementation
```

## Error Handling

### Agent Failure

If an agent crashes mid-task:

1. Item retains its assignee and `agent:*` label
2. No other agent will claim it (assignee check)
3. After 30 minutes with no status update, the stale claim detector clears the assignee
4. Item returns to its previous ready state for another agent to pick up
5. A `[SYSTEM]` comment records the stale claim clearance

### Budget Exhaustion

If token budget runs out during a session:

1. Agent commits current progress (if Coder)
2. Agent logs progress to GitHub comment
3. Agent removes its assignee claim and `agent:*` label
4. Agent exits gracefully
5. Next agent invocation picks up where it left off

### Merge Conflicts

When a PR has merge conflicts:

1. Auto-merge (or Reviewer) detects conflict, adds `status:blocked` and a comment
2. Coder polls for own PRs with `status:blocked`
3. Coder rebases the branch onto main, resolves conflicts
4. Coder pushes and sets `status:review` for re-review
5. If rebase is not possible, Coder adds `status:needs-human`
