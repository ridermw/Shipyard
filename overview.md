# Shipyard Skills Overview

This document provides an overview of the agent skills in Shipyard. Each skill is a Claude Code skill that gives Claude a specialized persona with specific responsibilities.

## Execution Model

**Key insight**: Each agent invocation is one-shot. External orchestration provides the loop.

```bash
# Each invocation handles one task or discovers no work
claude -p "$(cat skill.md) Poll for work, claim, execute, release." \
  --max-turns 20 \
  --max-budget-usd 5.00 \
  --allowedTools "Bash(gh *),Read,Edit,Write" \
  --no-session-persistence
```

See [orchestration.md](orchestration.md) for complete invocation patterns.

## Skills vs Hooks

| Mechanism | Purpose | Enforcement |
|-----------|---------|-------------|
| **Skills** | Provide persona and instructions | Soft — Claude follows but can deviate |
| **Hooks** | Block forbidden operations | Hard — tool call is rejected |

Skills tell agents *what* to do. Hooks *prevent* forbidden actions. See [hooks.md](hooks.md).

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
4. Terminate (via --max-turns limit)
```

**Note**: Agents do not "exit cleanly" on their own. The `--max-turns` flag forces termination. If an agent completes its task before the limit, it simply stops making tool calls and the session ends.

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

### Budget Enforcement

Budget is enforced externally via CLI flags, not internally by agents:

```bash
claude -p "..." \
  --max-turns 20 \           # Force termination after N turns
  --max-budget-usd 5.00      # Hard budget cap
```

| Agent | `--max-turns` | `--max-budget-usd` |
|-------|---------------|-------------------|
| PM | 10 | 2.00 |
| Coder | 20 | 5.00 |
| Reviewer | 15 | 3.00 |

If budget/turns exhausted mid-task:

1. Agent terminates immediately
2. Claim remains (assignee still set)
3. Stale detection (30 min) clears the claim
4. Next invocation picks up from GitHub state

There is no cross-session budget tracking. Cost management is handled externally (API dashboard, billing alerts, orchestration monitoring).

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

## CLI Invocation Examples

### PM Agent

```bash
claude -p "$(cat .shipyard/skills/pm/SKILL.md)

Poll GitHub Discussions in 'Planning' category with no status label.
If found: claim by adding status:planning, analyze, create Issues, close Discussion.
If none found: report no work available." \
  --max-turns 10 \
  --max-budget-usd 2.00 \
  --allowedTools "Bash(gh *)" \
  --no-session-persistence
```

### Coder Agent

```bash
claude -p "$(cat .shipyard/skills/coder/SKILL.md)

Poll GitHub Issues with status:ready label.
If found: claim via assignee, implement in worktree, create PR with status:review.
Also check for own PRs with status:changes-requested or status:blocked.
If none found: report no work available." \
  --max-turns 20 \
  --max-budget-usd 5.00 \
  --allowedTools "Bash(gh *),Bash(git *),Read,Edit,Write" \
  --no-session-persistence
```

### Reviewer Agent

```bash
claude -p "$(cat .shipyard/skills/reviewer/SKILL.md)

Poll GitHub PRs with status:review label (exclude self-authored).
If found: claim via assignee, review code, verify intent, approve or request changes.
If none found: report no work available." \
  --max-turns 15 \
  --max-budget-usd 3.00 \
  --allowedTools "Bash(gh *),Read" \
  --no-session-persistence
```

See [orchestration.md](orchestration.md) for complete orchestration patterns.
