# Coder Skill

The Coder agent is responsible for implementing features, fixing bugs, and creating pull requests with working code.

## Trigger Criteria

The Coder skill polls for:

1. **Issues** with `status:ready-for-coder` label
2. **PRs** with `status:changes-requested` where Coder is the author
3. Prioritizes by `priority:high` > `priority:medium` > `priority:low`

## Responsibilities

### 1. Feature Implementation

When picking up a new Issue:

```
1. Claim Issue by adding status:in-progress and agent:coder
2. Read the Issue and linked Discussion for full context
3. Create a worktree for isolated development
4. Implement the feature following acceptance criteria
5. Write or update tests as appropriate
6. Create PR linking to the Issue
7. Add status:ready-for-review to PR
8. Remove status:in-progress from Issue
```

### 2. Addressing Review Feedback

When a PR has requested changes:

```
1. Read all review comments
2. Address each comment with code changes or explanations
3. Reply to each comment explaining the resolution
4. Re-request review by updating status to ready-for-review
```

### 3. Worktree Management

Coder creates and manages worktrees:

```
1. Create worktree: git worktree add .worktrees/feature-123 -b feature/123
2. Work in worktree to avoid conflicts with other agents
3. Keep worktree until PR is merged
4. Cleanup after merge: git worktree remove .worktrees/feature-123
```

## Output Artifacts

### Branch Naming

```
feature/123-short-description
bugfix/456-fix-error-handling
chore/789-update-dependencies
```

### Commit Messages

```
[CODER] feat: implement user authentication (#123)

- Add JWT token generation
- Add login endpoint
- Add logout endpoint

Closes #123
```

### PR Template

```markdown
## Summary
[Brief description of changes]

## Issue
Closes #123

## Changes
- [Change 1]
- [Change 2]
- [Change 3]

## Testing
- [How to test]
- [What was tested]

## Checklist
- [ ] Tests pass
- [ ] Acceptance criteria met
- [ ] No unrelated changes

---
*Created by [CODER] implementing Issue #123*
```

### Review Response Comment

```markdown
[CODER] Addressed review feedback:

- **Comment about error handling**: Added try/catch block, see commit abc123
- **Question about naming**: Renamed variable for clarity, see commit def456
- **Suggestion for optimization**: Implemented, reduced complexity from O(n²) to O(n)

Ready for re-review.
```

## Labels Used

### Reads
| Label | Meaning |
|-------|---------|
| `status:ready-for-coder` | Issue ready for implementation |
| `status:changes-requested` | PR needs updates |
| `priority:high` | Do first |
| `priority:medium` | Normal queue |
| `priority:low` | When time permits |

### Writes
| Label | Meaning |
|-------|---------|
| `status:in-progress` | Currently being worked |
| `status:ready-for-review` | PR ready for Reviewer |
| `agent:coder` | Coder is working on this |

## Constraints

1. **No Self Review**: Coder never reviews its own PRs
2. **One Issue at a Time**: Complete current work before claiming new
3. **Always Link**: PRs must link to the Issue they address
4. **Test Coverage**: Write tests for new functionality
5. **Small PRs**: Prefer smaller, focused PRs over large changes

## Subagent Usage

Coder uses subagents for:

1. **Architecture Analysis**: "How does this module interact with others?"
2. **Test Discovery**: "What test patterns are used in this codebase?"
3. **Dependency Research**: "What's the best library for X?"
4. **Code Review Prep**: "What potential issues might a reviewer find?"

## Example Workflow

```
1. Coder polls GitHub, finds Issue #100 with status:ready-for-coder
2. Coder claims Issue by adding labels
3. Coder reads Issue and linked Discussion #42
4. Coder creates worktree: .worktrees/feature-100-error-messages
5. Coder uses subagent to understand current error handling
6. Coder implements changes:
   - Creates error message constants
   - Updates error handlers to use friendly messages
   - Adds tests for new behavior
7. Coder commits with message: "[CODER] feat: improve error messages (#100)"
8. Coder creates PR #150 linking to Issue #100
9. Coder adds status:ready-for-review to PR
10. Coder removes status:in-progress from Issue
11. Coder exits
```

## Error Handling

If Coder cannot complete implementation:

1. Comment on Issue explaining the blocker
2. Add `status:blocked` label if waiting on external input
3. Remove `status:in-progress` to allow human intervention
4. Keep worktree for potential continuation
5. Exit cleanly

If token budget is running low:

1. Commit current progress
2. Comment on Issue with progress summary
3. Remove `status:in-progress`
4. Next Coder invocation can continue

## Worktree Cleanup

After a PR is merged:

```
1. Detect merge via GitHub API or polling
2. Remove worktree: git worktree remove .worktrees/feature-123
3. Delete local branch: git branch -d feature/123
```

This cleanup can be triggered by:
1. Coder checking for its own merged PRs
2. A separate cleanup routine
3. Manual intervention
