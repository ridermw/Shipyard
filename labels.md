# GitHub Labels

This document defines the label taxonomy used by Shipyard to coordinate agent workflows. Labels serve as the "state machine" that routes work between agents.

## Label Categories

### Status Labels

These labels indicate the current state of work items. Only one status label should be active at a time.

| Label | Color | Description | Used On |
|-------|-------|-------------|---------|
| `status:needs-planning` | `#D4C5F9` | Awaiting PM to create spec | Discussions |
| `status:planning-in-progress` | `#BFD4F2` | PM is actively working | Discussions |
| `status:planned` | `#0E8A16` | PM completed, Issues created | Discussions |
| `status:ready-for-coder` | `#FBCA04` | Issue ready for implementation | Issues |
| `status:in-progress` | `#BFD4F2` | Work actively happening | Issues, PRs |
| `status:ready-for-review` | `#FBCA04` | PR awaiting Reviewer | PRs |
| `status:changes-requested` | `#E99695` | Reviewer requested changes | PRs |
| `status:approved` | `#0E8A16` | Reviewer approved | PRs |
| `status:merged` | `#6F42C1` | PR was merged | PRs |
| `status:rejected` | `#B60205` | PR was rejected by Owner | PRs |
| `status:blocked` | `#B60205` | Waiting on external factor | Issues, PRs |
| `status:needs-human` | `#D93F0B` | Requires human intervention | Any |
| `status:needs-split` | `#D93F0B` | PR scope too large | PRs |

### Priority Labels

Indicate urgency. Agents process higher priority items first.

| Label | Color | Description |
|-------|-------|-------------|
| `priority:high` | `#B60205` | Process immediately |
| `priority:medium` | `#FBCA04` | Normal queue |
| `priority:low` | `#0E8A16` | When time permits |

### Type Labels

Categorize the nature of the work.

| Label | Color | Description |
|-------|-------|-------------|
| `type:feature` | `#1D76DB` | New functionality |
| `type:bug` | `#B60205` | Defect to fix |
| `type:chore` | `#5319E7` | Maintenance, refactoring |
| `type:docs` | `#0075CA` | Documentation only |
| `type:question` | `#D876E3` | Needs clarification |

### Agent Labels

Track which agent is working on an item. Prevents duplicate work.

| Label | Color | Description |
|-------|-------|-------------|
| `agent:pm` | `#C2E0C6` | PM is working on this |
| `agent:coder` | `#C2E0C6` | Coder is working on this |
| `agent:reviewer` | `#C2E0C6` | Reviewer is working on this |
| `agent:owner` | `#C2E0C6` | Owner is evaluating |
| `agent:librarian` | `#C2E0C6` | Librarian is processing |

### Documentation Labels

Track documentation status for merged PRs.

| Label | Color | Description |
|-------|-------|-------------|
| `docs:updated` | `#0E8A16` | Documentation is current |
| `docs:pending` | `#FBCA04` | Doc PR created, not yet merged |
| `docs:needs-review` | `#D93F0B` | Librarian needs human help |

## Label Transitions

### Discussion Lifecycle

```
(created) → status:needs-planning
         → status:planning-in-progress (PM claims)
         → status:planned (PM completes)
```

### Issue Lifecycle

```
(created by PM) → status:ready-for-coder
               → status:in-progress (Coder claims)
               → (PR created, Issue remains in-progress)
               → (PR merged, Issue closed)
```

### PR Lifecycle

```
(created) → status:ready-for-review
         → status:changes-requested (Reviewer feedback)
         → status:ready-for-review (Coder updates)
         → status:approved (Reviewer approves)
         → status:merged (Owner merges)
         → docs:updated (Librarian completes)
```

## Setup Script

Create these labels in a new repository:

```bash
#!/bin/bash
# shipyard-labels.sh

REPO="owner/repo"

# Status labels
gh label create "status:needs-planning" --color "D4C5F9" --description "Awaiting PM spec" --repo $REPO
gh label create "status:planning-in-progress" --color "BFD4F2" --description "PM actively working" --repo $REPO
gh label create "status:planned" --color "0E8A16" --description "Planning complete" --repo $REPO
gh label create "status:ready-for-coder" --color "FBCA04" --description "Ready for implementation" --repo $REPO
gh label create "status:in-progress" --color "BFD4F2" --description "Work in progress" --repo $REPO
gh label create "status:ready-for-review" --color "FBCA04" --description "Awaiting review" --repo $REPO
gh label create "status:changes-requested" --color "E99695" --description "Reviewer requested changes" --repo $REPO
gh label create "status:approved" --color "0E8A16" --description "Approved by reviewer" --repo $REPO
gh label create "status:merged" --color "6F42C1" --description "PR merged" --repo $REPO
gh label create "status:rejected" --color "B60205" --description "PR rejected" --repo $REPO
gh label create "status:blocked" --color "B60205" --description "Blocked on external" --repo $REPO
gh label create "status:needs-human" --color "D93F0B" --description "Human intervention needed" --repo $REPO
gh label create "status:needs-split" --color "D93F0B" --description "PR scope too large" --repo $REPO

# Priority labels
gh label create "priority:high" --color "B60205" --description "Urgent" --repo $REPO
gh label create "priority:medium" --color "FBCA04" --description "Normal priority" --repo $REPO
gh label create "priority:low" --color "0E8A16" --description "When time permits" --repo $REPO

# Type labels
gh label create "type:feature" --color "1D76DB" --description "New feature" --repo $REPO
gh label create "type:bug" --color "B60205" --description "Bug fix" --repo $REPO
gh label create "type:chore" --color "5319E7" --description "Maintenance" --repo $REPO
gh label create "type:docs" --color "0075CA" --description "Documentation" --repo $REPO
gh label create "type:question" --color "D876E3" --description "Question" --repo $REPO

# Agent labels
gh label create "agent:pm" --color "C2E0C6" --description "PM working" --repo $REPO
gh label create "agent:coder" --color "C2E0C6" --description "Coder working" --repo $REPO
gh label create "agent:reviewer" --color "C2E0C6" --description "Reviewer working" --repo $REPO
gh label create "agent:owner" --color "C2E0C6" --description "Owner evaluating" --repo $REPO
gh label create "agent:librarian" --color "C2E0C6" --description "Librarian working" --repo $REPO

# Docs labels
gh label create "docs:updated" --color "0E8A16" --description "Docs current" --repo $REPO
gh label create "docs:pending" --color "FBCA04" --description "Doc PR pending" --repo $REPO
gh label create "docs:needs-review" --color "D93F0B" --description "Docs need help" --repo $REPO

echo "Labels created successfully!"
```

## Label Rules

### Mutual Exclusivity

Some labels are mutually exclusive:

1. Only one `status:*` label at a time (except `status:blocked` can coexist)
2. Only one `agent:*` label per agent type (can have `agent:coder` and `agent:reviewer` simultaneously if different stages)
3. Only one `priority:*` label at a time

### Required Labels

When creating items:

1. **Issues** must have: one `type:*`, one `priority:*`, one `status:*`
2. **PRs** must have: one `status:*`
3. **Discussions** should have: one `status:*` after PM processes

### Agent Claiming Rules

Before an agent claims work:

1. Check no other `agent:*` label exists for same agent type
2. Check the item has the expected `status:*` for that agent
3. Add the agent label atomically with status change
