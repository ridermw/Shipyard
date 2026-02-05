# Coder Skill

The Coder agent is responsible for implementing features, fixing bugs, creating pull requests with working code, updating documentation, and resolving merge conflicts.

## Trigger Criteria

The Coder skill polls for:

1. **Issues** with `status:ready` label (new work)
2. **PRs** with `status:changes-requested` where Coder is the author (review feedback)
3. **PRs** with `status:blocked` where Coder is the author (merge conflicts, CI failures)
4. Prioritizes by `priority:high` > `priority:medium` > `priority:low`, then oldest first

## Claiming

Coder uses assignee-based optimistic locking (see [labels.md](labels.md) for details):

1. Assign self via `gh issue edit N --add-assignee BOT_USERNAME`
2. Re-read the item to verify sole assignee
3. If conflict detected (multiple assignees), back off and try next item
4. On success, add `agent:coder` label and update status

For PRs with `status:changes-requested` or `status:blocked` where Coder is already the author, the Coder is already assigned -- no claim step needed.

## Responsibilities

### 1. Feature Implementation

When picking up a new Issue:

```
1. Claim Issue (assign self, verify, add status:in-progress + agent:coder)
2. Read the Issue and linked Discussion for full context
3. Create a worktree for isolated development
4. Implement the feature following acceptance criteria
5. Write or update tests as appropriate
6. Update documentation for user-facing changes (see Documentation section)
7. Create PR linking to the Issue
8. Add status:review to PR
```

### 2. Addressing Review Feedback

When a PR has requested changes:

```
1. Read all review comments
2. Address each comment with code changes or explanations
3. Reply to each comment explaining the resolution
4. Re-request review by updating status to status:review
```

### 3. Documentation

Coder is responsible for keeping docs in sync with code changes. Documentation updates are included in the same PR as the feature.

When implementing a feature, check:

- **User-facing behavior change?** Update relevant user docs, README sections
- **New or changed API endpoint?** Update API documentation
- **New or changed configuration?** Update config documentation
- **New feature?** Add to CHANGELOG.md under `[Unreleased]`

The PR checklist includes a documentation item that Reviewer will verify.

### 4. Merge Conflict Resolution

Coder polls for own PRs with `status:blocked`:

```
1. Check if the block is a merge conflict or CI failure
2. For merge conflicts:
   a. VERIFY OWNERSHIP: Confirm Coder is still assigned to the PR
   b. Rebase the branch onto main
   c. Resolve conflicts
   d. Force-push the rebased branch (after ownership check)
   e. Update status to status:review for re-review
3. For CI failures:
   a. Read CI logs to identify the failure
   b. Fix the issue
   c. Push fix commit
   d. Update status to status:review for re-review
4. If unable to resolve: add status:needs-human and comment explaining the blocker
```

**Force-push safety**: Before force-pushing, Coder must verify it still owns the PR. Another agent may have taken over during a stale claim. See [hooks.md](hooks.md) for hook-based enforcement.

### 5. Worktree Management

Coder creates and manages worktrees:

```
1. Create worktree: git worktree add .worktrees/feature-123 -b feature/123
2. Work in worktree to avoid conflicts with other agents
3. Keep worktree until PR is merged
4. Cleanup after merge: git worktree remove .worktrees/feature-123
```

**Worktree collision prevention**: Two Coders cannot create the same worktree because the claiming mechanism prevents duplicate work on the same Issue. If worktree already exists:

```
1. Check if worktree is for this Issue (naming convention)
2. If yes: reuse existing worktree (resuming previous work)
3. If naming conflict: use Issue number suffix (.worktrees/feature-123-retry)
```

**Orphan worktree cleanup**: At session start, Coder can check for orphan worktrees (merged PRs) and clean them up.

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
- [ ] Documentation updated for user-facing changes

---
*Created by [CODER] implementing Issue #123*
```

### Review Response Comment

```markdown
[CODER] Addressed review feedback:

- **Comment about error handling**: Added try/catch block, see commit abc123
- **Question about naming**: Renamed variable for clarity, see commit def456
- **Suggestion for optimization**: Implemented, reduced complexity from O(n^2) to O(n)

Ready for re-review.
```

## Labels Used

### Reads
| Label | Meaning |
|-------|---------|
| `status:ready` | Issue ready for implementation |
| `status:changes-requested` | PR needs updates |
| `status:blocked` | PR has merge conflict or CI failure |
| `priority:high` | Do first |
| `priority:medium` | Normal queue |
| `priority:low` | When time permits |

### Writes
| Label | Meaning |
|-------|---------|
| `status:in-progress` | Currently being worked |
| `status:review` | PR ready for Reviewer |
| `status:needs-human` | Cannot resolve, needs human |
| `agent:coder` | Coder is working on this |

## Constraints

1. **No Self-Review**: Coder never reviews its own PRs
2. **One Issue at a Time**: Complete current work before claiming new
3. **Always Link**: PRs must link to the Issue they address
4. **Test Coverage**: Write tests for new functionality
5. **Small PRs**: Prefer smaller, focused PRs over large changes
6. **Docs in PR**: Include documentation updates in the feature PR

## Subagent Usage

Coder uses subagents for:

1. **Architecture Analysis**: "How does this module interact with others?"
2. **Test Discovery**: "What test patterns are used in this codebase?"
3. **Dependency Research**: "What's the best library for X?"
4. **Code Review Prep**: "What potential issues might a reviewer find?"
5. **Doc Discovery**: "What documentation files reference this module?"

## Example Workflow

### New Feature

```
1. Coder polls GitHub, finds Issue #100 with status:ready
2. Coder claims Issue (assign self, verify, add labels)
3. Coder reads Issue and linked Discussion #42
4. Coder creates worktree: .worktrees/feature-100-error-messages
5. Coder uses subagent to understand current error handling
6. Coder implements changes:
   - Creates error message constants
   - Updates error handlers to use friendly messages
   - Adds tests for new behavior
   - Updates README with error code reference
7. Coder commits with message: "[CODER] feat: improve error messages (#100)"
8. Coder creates PR #150 linking to Issue #100
9. Coder adds status:review to PR
10. Coder releases claim on Issue (remove assignee, remove agent:coder)
11. Coder exits
```

### Merge Conflict Resolution

```
1. Coder polls GitHub, finds own PR #150 with status:blocked
2. Coder reads [SYSTEM] comment: "Merge blocked: conflicts with main"
3. Coder rebases feature branch onto main
4. Coder resolves conflicts
5. Coder force-pushes rebased branch
6. Coder updates PR status to status:review
7. Coder exits
```

## Error Handling

If Coder cannot complete implementation:

1. Comment on Issue explaining the blocker
2. Add `status:blocked` label if waiting on external input
3. Release claim (remove assignee, remove `agent:coder`)
4. Keep worktree for potential continuation
5. Exit cleanly

If token budget is running low:

1. Commit current progress
2. Comment on Issue with progress summary
3. Release claim
4. Next Coder invocation can continue

## Budget

Budget is enforced externally via CLI flags:

```bash
claude -p "..." \
  --max-turns 20 \
  --max-budget-usd 5.00
```

If budget/turns exhausted mid-implementation:
1. Agent terminates immediately
2. Claim remains (assignee still set)
3. Partial work may exist in worktree (uncommitted) or branch (committed)
4. Stale detection (30 min) clears the claim
5. Next Coder invocation can resume from worktree/branch state

There is no cross-session tracking. See [orchestration.md](orchestration.md).

## Worktree Cleanup

After a PR is merged:

```
1. Detect merge via GitHub API or polling
2. Remove worktree: git worktree remove .worktrees/feature-123
3. Delete local branch: git branch -d feature/123
```

This cleanup can be triggered by:
1. Coder checking for its own merged PRs at the start of a session
2. `sy worktrees clean` command
3. Manual intervention
