# Shipyard Skills Overview

This document provides an overview of the agent skills in Shipyard. Each skill is a Claude Code skill that gives Claude a specialized persona with specific responsibilities.

## Skill Architecture

All Shipyard skills follow a common pattern:

### 1. Polling Phase

```
1. Query GitHub for items matching agent's criteria
2. Filter out items already claimed (have an assignee)
3. Filter out items the agent created (no self-approval)
4. Select highest priority unclaimed item
5. If no work found, exit gracefully
```

### 2. Claim Phase

Shipyard uses assignee-based optimistic locking to prevent duplicate work:

```
1. Assign self to the item via gh issue edit N --add-assignee BOT_USERNAME
2. Re-read the item to verify sole assignee
3. If conflict detected (multiple assignees): back off, remove self, try next item
4. On success: add agent label (e.g., agent:coder) and update status label
5. Leave comment announcing work has begun
```

Label-based claiming has a TOCTOU race condition (GitHub label operations are not atomic), so Shipyard uses assignees instead.

### 3. Work Phase

```
1. Read relevant context (Discussion, Issue, existing code)
2. Use subagents for complex analysis without polluting context
3. Perform the specialized task
4. Track token usage against per-session cap
```

### 4. Completion Phase

```
1. Update GitHub with results (PR, comments, label changes)
2. Remove assignee claim and agent label
3. Leave summary comment explaining what was done
4. Exit cleanly
```

## Common Skill Elements

### GitHub CLI Usage

All skills use the `gh` CLI for GitHub operations:

```bash
# List issues by label
gh issue list --label "status:ready"

# Claim an item (assign self)
gh issue edit 123 --add-assignee bot-username

# Add a label
gh issue edit 123 --add-label "status:in-progress"

# Create a PR
gh pr create --title "..." --body "..."

# Add a comment
gh issue comment 123 --body "[AGENT] Comment text"
```

### Agent Identity Prefix

All agent communications include a prefix identifying which agent wrote it:

| Agent | Prefix |
|-------|--------|
| PM | `[PM]` |
| Coder | `[CODER]` |
| Reviewer | `[REVIEWER]` |

System-generated messages (stale claim clearance, merge failures) use `[SYSTEM]`.

### Budget Awareness

Each skill enforces a per-session token cap:

```yaml
# .shipyard/config.yaml
budget:
  session_max_tokens: 100000
```

```
On startup:
  Read session_max_tokens from config

During work:
  Periodically check usage against cap
  If approaching cap:
    Save progress to GitHub comment
    Release claim (remove assignee, remove agent label)
    Exit gracefully
```

There is no cross-session budget tracking. Each invocation is independent. Cost management across sessions is handled externally (API dashboard, billing alerts).

### Subagent Usage

When analysis would pollute the main context, skills spawn subagents:

```
Main agent: "Analyze the auth module for security concerns"
  └─▶ Subagent reads files, performs analysis
  └─▶ Returns: "Found 3 concerns: [summary]"
Main agent continues with only the summary in context
```

## Skill Files

| Skill | File | Documentation |
|-------|------|---------------|
| PM | `shipyard-pm` | [pm.md](pm.md) |
| Coder | `shipyard-coder` | [coder.md](coder.md) |
| Reviewer | `shipyard-reviewer` | [reviewer.md](reviewer.md) |

Merge logic is handled by branch protection rules and an optional script. See [merge.md](merge.md).

## Installation Location

Skills can be installed:

1. **Globally**: In Claude Code's skill directory for use across all repos
2. **Per Repo**: In `.shipyard/skills/` for repo-specific customization

The `sy update` command synchronizes skills from the Shipyard repository.

## Customization

Skills can be customized by:

1. **Configuration**: Adjusting `.shipyard/config.yaml` settings
2. **Override**: Placing modified skill files in `.shipyard/skills/`
3. **Extension**: Creating new agent types following the same patterns
