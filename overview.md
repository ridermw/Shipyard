# Shipyard Skills Overview

This document provides an overview of the agent skills in Shipyard. Each skill is a Claude Code skill that gives Claude a specialized persona with specific responsibilities.

## Skill Architecture

All Shipyard skills follow a common pattern:

### 1. Polling Phase

```
1. Query GitHub for items matching agent's criteria
2. Filter out items already claimed by another agent
3. Filter out items the agent created (no self approval)
4. Select highest priority unclaimed item
5. If no work found, exit gracefully
```

### 2. Claim Phase

```
1. Add agent label (e.g., agent:coder)
2. Add in progress label if applicable
3. Leave comment announcing work has begun
```

### 3. Work Phase

```
1. Read relevant context (Discussion, Issue, existing code)
2. Use subagents for complex analysis without polluting context
3. Perform the specialized task
4. Track token usage against budget
```

### 4. Completion Phase

```
1. Update GitHub with results (PR, comments, label changes)
2. Remove in progress label
3. Leave summary comment explaining what was done
4. Exit cleanly
```

## Common Skill Elements

### GitHub CLI Usage

All skills use the `gh` CLI for GitHub operations:

```bash
# List discussions needing planning
gh api graphql -f query='...'

# List issues by label
gh issue list --label "status:ready-for-coder"

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
| Owner | `[OWNER]` |
| Librarian | `[LIBRARIAN]` |

### Budget Awareness

Each skill checks token usage at startup and periodically during execution:

```
If usage > 80% of budget:
  Log warning to GitHub comment
  Exit without processing new work

If usage approaching 80% during work:
  Save progress to GitHub comment
  Remove in progress claim
  Exit gracefully
```

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
| Owner | `shipyard-owner` | [owner.md](owner.md) |
| Librarian | `shipyard-librarian` | [librarian.md](librarian.md) |

## Installation Location

Skills can be installed:

1. **Globally**: In Claude Code's skill directory for use across all repos
2. **Per Repo**: In `.shipyard/skills/` for repo specific customization

The `sy update` command synchronizes skills from the Shipyard repository.

## Customization

Skills can be customized by:

1. **Configuration**: Adjusting `.shipyard/config.yaml` settings
2. **Override**: Placing modified skill files in `.shipyard/skills/`
3. **Extension**: Creating new agent types following the same patterns
