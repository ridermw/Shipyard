# Merge Logic

This document specifies how approved PRs are merged in Shipyard. Once a PR has an approving review and CI passes, merging is a deterministic action that does not require LLM reasoning. Shipyard delegates this to GitHub branch protection rules and an optional auto-merge script.

## Branch Protection Requirements

Configure these settings on the `main` branch:

| Setting | Value | Purpose |
|---------|-------|---------|
| Require pull request reviews | 1 approval | Ensures Reviewer has approved |
| Require status checks to pass | Yes | Ensures CI is green |
| Require branches to be up to date | Yes | Prevents merge conflicts |
| Require linear history | Yes | Clean squash-merge history |
| Restrict force pushes | Yes | Protect main branch |
| Restrict deletions | Yes | Protect main branch |

## Auto-Merge Behavior

### GitHub Native Auto-Merge

The simplest approach: enable GitHub's auto-merge feature. When a PR is approved and CI passes, GitHub merges it automatically.

```bash
# Enable auto-merge on a PR
gh pr merge 150 --auto --squash
```

Coder can enable auto-merge when creating the PR. Once Reviewer approves and CI passes, GitHub handles the rest.

### `sy merge` Script (Alternative)

For more control, the `sy merge` command checks merge eligibility and executes:

```
1. Find PRs with status:approved label
2. For each PR:
   a. Verify at least 1 approving review exists
   b. Verify all required CI status checks pass
   c. Verify no merge conflicts
   d. If all pass: squash merge
   e. If conflicts: add status:blocked, comment with reason
   f. If CI failing: add status:blocked, comment with reason
3. Linked Issues are auto-closed via commit message (Closes #N)
```

### Merge Failure Handling

| Failure | Action |
|---------|--------|
| Merge conflicts | Add `status:blocked` label, comment `[SYSTEM] Merge blocked: conflicts with main. Coder: please rebase.` |
| CI failing | Add `status:blocked` label, comment `[SYSTEM] Merge blocked: CI checks failing.` |
| Review revoked | Remove `status:approved`, add `status:review` |
| Branch deleted | Comment `[SYSTEM] Source branch no longer exists.`, add `status:needs-human` |

### Post-Merge

After successful merge:

1. PR is marked as merged (GitHub native state)
2. Linked Issues are auto-closed via `Closes #N` in the squash commit message
3. Worktree cleanup can be triggered by Coder's next invocation or `sy worktrees clean`
4. Documentation was included by Coder in the feature PR

## Configuration

In `.shipyard/config.yaml`:

```yaml
merge:
  strategy: squash          # squash, merge, or rebase
  auto_merge: true          # Enable GitHub auto-merge on PR creation
  delete_branch: true       # Delete source branch after merge
```

## Setup

```bash
# Configure branch protection via gh CLI
gh api repos/{owner}/{repo}/branches/main/protection -X PUT \
  -f required_status_checks='{"strict":true,"contexts":["ci"]}' \
  -f required_pull_request_reviews='{"required_approving_review_count":1}' \
  -f enforce_admins=true \
  -f restrictions=null
```

Or configure via GitHub Settings > Branches > Branch protection rules.
