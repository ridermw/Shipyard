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
│                    Labels + Comments                            │
│                    (State Machine)                              │
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
                           ▼
                   ┌───────────────┐
                   │  Git Worktree │
                   │   (Isolated)  │
                   └───────────────┘
```

## Data Flow

### State Transitions via Labels

Labels encode the workflow state machine. Agents read labels to find work and write labels to signal state changes.

```
Discussion Created
       │
       ▼
[status:needs-planning] ──PM──▶ [status:planned]
                                      │
                                      ▼
                              Issue Created
                                      │
                                      ▼
                        [status:ready-for-coder]
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
                        [status:ready-for-review]
                                      │
                              ──Reviewer──
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
        [status:changes-requested]            [status:approved]
                    │                                   │
             ──Coder──                           ──Owner──
                    │                                   │
                    ▼                                   ▼
        [status:ready-for-review]                  Merged
                                                       │
                                                ──Librarian──
                                                       │
                                                       ▼
                                                 Docs Updated
```

### Agent Polling Patterns

Each agent polls for specific label combinations:

| Agent | Polls For | Claims With |
|-------|-----------|-------------|
| PM | `status:needs-planning` | `status:planning-in-progress` |
| Coder | `status:ready-for-coder` | `status:in-progress`, `agent:coder` |
| Reviewer | `status:ready-for-review` AND NOT `author:self` | `agent:reviewer` |
| Owner | `status:approved` | `agent:owner` |
| Librarian | Recently merged PRs | `agent:librarian` |

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
│   ├── shipyard-reviewer/
│   ├── shipyard-owner/
│   └── shipyard-librarian/
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

If an agent crashes mid task:

1. Work item retains `in-progress` label
2. No other agent will claim it (label based exclusion)
3. Human reviews stale items and either:
   a. Removes label to allow retry
   b. Manually completes the work
   c. Closes the item

### Budget Exhaustion

If token budget approaches 80%:

1. Agent logs current progress to GitHub comment
2. Agent removes its `in-progress` claim
3. Agent exits gracefully
4. Next agent invocation continues the work

### Conflict Resolution

If two agents claim the same work (race condition):

1. Second agent detects existing `agent:*` label
2. Second agent backs off and finds other work
3. GitHub's atomic label operations prevent true conflicts
