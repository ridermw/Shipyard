# GitHub Labels

This document defines the label taxonomy used by Shipyard to coordinate agent workflows. Labels serve as the state machine that routes work between agents.

## Label Categories

### Status Labels (8)

These labels indicate the current state of work items. Only one status label should be active at a time.

| Label | Color | Description | Used On |
|-------|-------|-------------|---------|
| `status:planning` | `#BFD4F2` | PM is actively planning this item | Discussions |
| `status:ready` | `#FBCA04` | Ready for Coder to implement | Issues |
| `status:in-progress` | `#BFD4F2` | Work actively happening | Issues, PRs |
| `status:review` | `#FBCA04` | PR awaiting Reviewer | PRs |
| `status:changes-requested` | `#E99695` | Reviewer requested changes | PRs |
| `status:approved` | `#0E8A16` | Reviewer approved, CI passed, ready to merge | PRs |
| `status:blocked` | `#B60205` | Waiting on external factor (CI failure, merge conflict, dependency) | Issues, PRs |
| `status:needs-human` | `#D93F0B` | Requires human intervention | Any |

### Agent Labels (3)

Informational labels indicating which agent is working on an item. The actual lock mechanism uses the assignee field (see Claiming Mechanism below).

| Label | Color | Description |
|-------|-------|-------------|
| `agent:pm` | `#C2E0C6` | PM is working on this |
| `agent:coder` | `#C2E0C6` | Coder is working on this |
| `agent:reviewer` | `#C2E0C6` | Reviewer is working on this |

### Priority Labels (3)

Indicate urgency. Agents process higher priority items first.

| Label | Color | Description |
|-------|-------|-------------|
| `priority:high` | `#B60205` | Process immediately |
| `priority:medium` | `#FBCA04` | Normal queue |
| `priority:low` | `#0E8A16` | When time permits |

### Type Labels (Optional)

Categorize the nature of the work. These are metadata, not part of the core workflow.

| Label | Color | Description |
|-------|-------|-------------|
| `type:feature` | `#1D76DB` | New functionality |
| `type:bug` | `#B60205` | Defect to fix |
| `type:chore` | `#5319E7` | Maintenance, refactoring |

## Claiming Mechanism

### Why Not Labels?

GitHub label operations are not atomic. Two agents polling simultaneously can both read "no `agent:*` label" and both attempt to claim the same item (TOCTOU race condition). Labels provide no mutual exclusion guarantee.

### Assignee-Based Optimistic Locking

Shipyard uses the Issue/PR `assignees` field as the locking mechanism:

1. **Attempt claim**: Agent assigns itself via `gh issue edit N --add-assignee BOT_USERNAME`
2. **Verify claim**: Agent re-reads the item and checks the assignee list
3. **Conflict detection**: If multiple assignees exist, the agent backs off (removes itself, waits, retries or moves on)
4. **Informational label**: After successful claim, agent adds the `agent:*` label and updates the status label
5. **Release**: On completion, agent removes itself as assignee, removes `agent:*` label, and updates status

This approach works because:
- Assignee writes are last-writer-wins, and the re-read step detects conflicts
- The window between assign and verify is small (milliseconds)
- In practice, agents poll on different schedules, making true races rare
- The worst case (both agents back off) is safe, just wasteful

### Stale Claim Detection

If an agent crashes or exceeds its token budget mid-task, the claim becomes stale:

- **Detection**: Any item with an assignee but no status update for 30 minutes is considered stale
- **Resolution**: The `sy` CLI (or a cron job) removes the assignee, removes the `agent:*` label, and returns the item to its previous ready state (`status:ready`, `status:review`, etc.)
- **Comment**: A `[SYSTEM]` comment is added noting the stale claim was cleared

## Label Transitions

### Discussion Lifecycle

```
Discussion created (no status label)
       │
       ▼
[status:planning] ──PM plans──▶ Discussion closed/answered
                                       │
                                       ▼
                               Issue(s) created with [status:ready]
```

PM polls for Discussions in the "Planning" category that have no `status:*` label. When PM claims a Discussion, it adds `status:planning`. When planning is complete, PM closes/answers the Discussion and creates Issues with `status:ready`.

### Issue Lifecycle

```
Issue created with [status:ready]
       │
       ▼
[status:in-progress] ──Coder implements──▶ PR created with [status:review]
                                            Issue remains open, linked to PR
                                            │
                                            ▼
                                     (PR merged → Issue auto-closed)
```

### PR Lifecycle

```
PR created with [status:review]
       │
       ▼
  ──Reviewer──
       │
       ├──▶ [status:approved] ──auto-merge──▶ Merged (GitHub tracks natively)
       │
       └──▶ [status:changes-requested] ──Coder fixes──▶ [status:review]
                                         (max 3 cycles, then status:needs-human)
```

### Recovery Paths

```
[status:blocked] ──48h timeout──▶ [status:needs-human]

[status:changes-requested] ──3 review cycles──▶ [status:needs-human]

PR rejected by Reviewer ──▶ PR closed, linked Issue returned to [status:ready]
                             (or [status:needs-human] if fundamentally misaligned)

Agent crash ──30min stale──▶ Claim cleared, item returns to previous ready state

Merge conflict ──▶ [status:blocked] ──Coder rebases──▶ [status:review]

CI failure ──▶ [status:blocked] ──Coder fixes──▶ [status:review]
```

## Setup Script

Create the 14 core labels in a new repository:

```bash
#!/bin/bash
# shipyard-labels.sh

REPO="owner/repo"

# Status labels
gh label create "status:planning" --color "BFD4F2" --description "PM actively planning" --repo $REPO
gh label create "status:ready" --color "FBCA04" --description "Ready for implementation" --repo $REPO
gh label create "status:in-progress" --color "BFD4F2" --description "Work in progress" --repo $REPO
gh label create "status:review" --color "FBCA04" --description "Awaiting review" --repo $REPO
gh label create "status:changes-requested" --color "E99695" --description "Reviewer requested changes" --repo $REPO
gh label create "status:approved" --color "0E8A16" --description "Approved, ready to merge" --repo $REPO
gh label create "status:blocked" --color "B60205" --description "Blocked on external factor" --repo $REPO
gh label create "status:needs-human" --color "D93F0B" --description "Human intervention needed" --repo $REPO

# Agent labels
gh label create "agent:pm" --color "C2E0C6" --description "PM working" --repo $REPO
gh label create "agent:coder" --color "C2E0C6" --description "Coder working" --repo $REPO
gh label create "agent:reviewer" --color "C2E0C6" --description "Reviewer working" --repo $REPO

# Priority labels
gh label create "priority:high" --color "B60205" --description "Urgent" --repo $REPO
gh label create "priority:medium" --color "FBCA04" --description "Normal priority" --repo $REPO
gh label create "priority:low" --color "0E8A16" --description "When time permits" --repo $REPO

echo "Created 14 labels successfully!"
```

Optionally, create type labels for metadata:

```bash
# Type labels (optional)
gh label create "type:feature" --color "1D76DB" --description "New feature" --repo $REPO
gh label create "type:bug" --color "B60205" --description "Bug fix" --repo $REPO
gh label create "type:chore" --color "5319E7" --description "Maintenance" --repo $REPO
```

## Label Rules

### Mutual Exclusivity

1. Only one `status:*` label at a time
2. Only one `agent:*` label at a time (one agent works on an item at a time)
3. Only one `priority:*` label at a time

### Required Labels

When creating items:

1. **Issues** must have: one `priority:*`, one `status:*`
2. **PRs** must have: one `status:*`
3. **Discussions** receive a `status:*` label when PM begins planning

### Type Labels

Type labels (`type:feature`, `type:bug`, `type:chore`) are optional metadata. They help with filtering and reporting but do not affect the workflow state machine.
